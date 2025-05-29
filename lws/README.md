# LWS Demos

- [LWS Demos](#lws-demos)
  - [前置准备](#前置准备)
    - [安装 LWS](#安装-lws)
    - [k8s 环境](#k8s-环境)
    - [下载模型](#下载模型)
  - [模型选择](#模型选择)
  - [LWS 多机推理 demo](#lws-多机推理-demo)
  - [HPA](#hpa)
    - [前置准备](#前置准备-1)
    - [ServiceMonitor 暴露 vllm 指标](#servicemonitor-暴露-vllm-指标)
    - [创建 HorizontalPodAutoscaler](#创建-horizontalpodautoscaler)
  - [检测](#检测)
    - [安装 vllm benchmark demo](#安装-vllm-benchmark-demo)
  - [PD 分离 Demo](#pd-分离-demo)
    - [安装 LMCache](#安装-lmcache)
    - [加载 pd proxy 脚本](#加载-pd-proxy-脚本)
    - [异构 PD 分离 demo](#异构-pd-分离-demo)
    - [同构 PD 分离 demo](#同构-pd-分离-demo)

## 前置准备

### 安装 LWS

```bash
CHART_VERSION=0.6.1
helm install lws oci://registry.k8s.io/lws/charts/lws \
  --version=$CHART_VERSION \
  --namespace lws-system \
  --create-namespace \
  --wait --timeout 300s
```

### k8s 环境

```bash
kubectl create ns demo-lws

# 下载demo 仓库
git clone git@github.com:nicole-lihui/kubecon-docs.git

# 国内加速
git clone https://gh-proxy.ygxz.in/https://github.com/nicole-lihui/kubecon-docs
```

### 下载模型

1. 本地下载，通过 host path 挂载到 pod 中

```bash
pip install modelscope

# 以 DeepSeek-R1-Distill-Qwen-1.5B 为例子 https://modelscope.cn/models/deepseek-ai/DeepSeek-R1-Distill-Qwen-32B
modelscope download --model deepseek-ai/DeepSeek-R1-Distill-Qwen-32B
```

当前方式在 k8s 环境中比较复杂，建议创建 pod 挂载 share pvc 方式存储

2. dataset 方式，通过 pod 挂载到 pvc 中（下面 demo 采用当前方案）

采用 [BaizeAI/dataset](https://github.com/BaizeAI/dataset) 项目来提前下载模型

```bash
kubectl -n demo-lws apply -f ./lws/datasets/deepseek-r1-distill-qwen-32b.yaml
```

## 模型选择

```
MODEL=deepseek-r1-distill-qwen-1-5b
```

## LWS 多机推理 demo

[/vllm-workspace/examples/online_serving/multi-node-serving.sh](https://github.com/vllm-project/vllm/blob/main/examples/online_serving/multi-node-serving.sh) 是 vllm 官方多机推理示例：

主要是通 ray 进行多机推理，当前脚本增加工作集群加入等待逻辑和参数检测。

**leader 运行命令**

```yaml
- name: vllm-leader
  image: docker.1ms.run/lmcache/vllm-openai:2025-04-18
  command:
    - sh
    - -c
  args:
    - |
      bash /vllm-workspace/examples/online_serving/multi-node-serving.sh leader \
      --ray_cluster_size=$(LWS_GROUP_SIZE); \
      python3 -m vllm.entrypoints.openai.api_server --port 8000 \
      --model /data/serving-model \
      --served-model-name  deepseek-r1-distill-qwen-1-5b \
      --tensor-parallel-size 2 \
      --pipeline_parallel_size 1
```

**worker 运行**

```yaml
- name: vllm-worker
  image: docker.1ms.run/lmcache/vllm-openai:2025-04-18
  command:
    - sh
    - -c
  args:
    - |
      bash /vllm-workspace/examples/online_serving/multi-node-serving.sh worker \
      --ray_address=$(LWS_LEADER_ADDRESS)
```

````bash
# install
kubectl -n demo-lws apply -f "./lws/demo/multi-${MODEL}.yaml"
```

uninstall

```bash
kubectl -n demo-lws delete -f "./lws/demo/multi-${MODEL}.yaml"
````

## HPA

### 前置准备

安装 prometheus 和 prometheus-adapter
可以参考 [vllm-project/production-stack](https://github.com/vllm-project/production-stack/blob/main/observability/README.md)

prometheus-adapter rules 自定义 hpa 指标

```yaml
rules:
  custom:
    - metricsQuery: sum by(namespace) (vllm:num_requests_waiting)
      name:
        as: vllm_num_requests_waiting
        matches: ""
      resources:
        overrides:
          namespace:
            resource: namespace
      seriesQuery: '{__name__=~"^vllm:num_requests_waiting$"}'
```

```bash
# 检测 hpa 指标是否生效
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq | grep vllm_num_requests_waiting -C 10
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/demo-lws/metrics/vllm_num_requests_waiting | jq
```

### ServiceMonitor 暴露 vllm 指标

```bash
kubectl -n demo-lws apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    operator.insight.io/managed-by: insight
  name: demo-lws-sm-metrics
spec:
  endpoints:
    - port: "vllm-service-port"
      interval: 15s
      path: /metrics
  namespaceSelector:
    matchNames:
      - demo-lws
  selector:
    matchLabels:
      prometheus-discovery: "true"
EOF
```

### 创建 HorizontalPodAutoscaler

```bash
kubectl -n demo-lws apply -f "./lws/hpa/${MODEL}.yaml"
```

uninstall

```
kubectl -n demo-lws delete -f "./lws/hpa/${MODEL}.yaml"
```

## 检测

### 安装 vllm benchmark demo

```bash
kubectl -n demo-lws apply -f ./lws/demo/vllm-benchmark.yaml

# 进入容器
kubectl -n demo-lws exec -it $(kubectl -n demo-lws get pod -l app=vllm-benchmark -o jsonpath='{.items[0].metadata.name}') bash
```

vllm benchmark 容器中执行

```bash
MODEL=deepseek-r1-distill-qwen-1-5b

PORT=8000
BASE_URL="http://${MODEL}:${PORT}"

curl --location "${BASE_URL}/v1/completions" \
  --header 'Content-Type: application/json' \
  --data "$(cat <<EOF
{
  "model": "${MODEL}",
  "prompt": "hi,"
}
EOF
)"
```

```bash
# 下载测试数据集
git clone https://www.modelscope.cn/datasets/otavia/ShareGPT_Vicuna_unfiltered.git

export MODEL=deepseek-r1-distill-qwen-1-5b
export DATASET_PATH=/vllm-workspace/ShareGPT_Vicuna_unfiltered/ShareGPT_V3_unfiltered_cleaned_split.json

python3 /vllm-workspace/benchmarks/benchmark_serving.py \
  --backend vllm \
  --model /data/serving-model \
  --served-model-name "$MODEL" \
  --host "$MODEL" \
  --port 8000 \
  --dataset-name random \
  --random-input 1024 \
  --random-output 512 \
  --random-range-ratio 0.8 \
  --dataset-path "$DATASET_PATH" \
  --max-concurrency 300 \
  --num-prompts 64
```

## PD 分离 Demo

### 安装 LMCache

```bash
kubectl -n demo-lws apply -f ./lws/demo/pd-lmcacheserve.yaml
```

### 加载 pd proxy 脚本

PD 分离需要有一个核心 router 进行 pd 的路由，因此这里采用 vllm example 的 python 脚本进行 demo
[disagg_proxy_server.py](https://github.com/vllm-project/vllm/blob/main/examples/others/lmcache/disagg_prefill_lmcache_v1/disagg_proxy_server.py)

暂时通过 configmap 方式挂载脚本，只做 demo

```bash
kubectl -n demo-lws apply -f ./lws/demo/pd-disagg-proxy-server.yaml
```

```bash
kubectl -n demo-lws delete -f ./lws/demo/pd-disagg-proxy-server.yaml
```

### 异构 PD 分离 demo

```bash
kubectl -n demo-lws apply -f ./lws/demo/pd-kvdiff-${MODEL}.yaml
```

uninstall

```bash
kubectl -n demo-lws delete -f ./lws/demo/pd-kvdiff-${MODEL}.yaml
```

检测

```bash
MODEL=pd-diff-deepseek-r1-distill-qwen-1-5b

PORT=8000
BASE_URL="http://${MODEL}:${PORT}"

curl --location "${BASE_URL}/v1/completions" \
  --header 'Content-Type: application/json' \
  --data "$(cat <<EOF
{
  "model": "${MODEL}",
  "prompt": "hi, what is vllm ?"
}
EOF
)"
```

### 同构 PD 分离 demo

```bash
kubectl -n demo-lws apply -f ./lws/demo/pd-kvboth-${MODEL}.yaml
```

uninstall

```bash
kubectl -n demo-lws delete -f ./lws/demo/pd-kvboth-${MODEL}.yaml
```

