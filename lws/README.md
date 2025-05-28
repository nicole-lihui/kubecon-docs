# LWS Demos

- [LWS Demos](#lws-demos)
  - [前置准备](#前置准备)
    - [安装 LWS](#安装-lws)
    - [k8s 环境](#k8s-环境)
    - [下载模型](#下载模型)
  - [LWS 多机推理 demo](#lws-多机推理-demo)
  - [HPA](#hpa)
    - [前置准备](#前置准备-1)
    - [ServiceMonitor 暴露 vllm 指标](#servicemonitor-暴露-vllm-指标)
    - [创建 HorizontalPodAutoscaler](#创建-horizontalpodautoscaler)
  - [检测](#检测)
    - [安装 vllm benchmark demo](#安装-vllm-benchmark-demo)
  - [PD 分离 Demo](#pd-分离-demo)
    - [安装 LMCache](#安装-lmcache)
    - [同构 PD 分离 demo](#同构-pd-分离-demo)
    - [异构 PD 分离 demo](#异构-pd-分离-demo)

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

## LWS 多机推理 demo

```bash
# install
MODEL=deepseek-r1-distill-qwen-1-5b
kubectl -n demo-lws apply -f "./lws/demo/multi-${MODEL}.yaml"

# uninstall

MODEL=deepseek-r1-distill-qwen-1-5b
kubectl -n demo-lws delete -f "./lws/demo/multi-${MODEL}.yaml"
```

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
MODEL=deepseek-r1-distill-qwen-1-5b
kubectl -n demo-lws apply -f "./lws/hpa/${MODEL}.yaml"
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

### 同构 PD 分离 demo
```bash
kubectl -n demo-lws apply -f ./lws/demo/pd-kvboth-deepseek-r1-distill-qwen-14b.yaml
```

uninstall
```bash
kubectl -n demo-lws delete -f ./lws/demo/pd-kvboth-deepseek-r1-distill-qwen-14b.yaml
```

### 异构 PD 分离 demo
```bash
kubectl -n demo-lws apply -f ./lws/demo/pd-kvdiff-deepseek-r1-distill-qwen-14b.yaml
```

uninstall

```bash
kubectl -n demo-lws delete -f ./lws/demo/pd-kvdiff-deepseek-r1-distill-qwen-14b.yaml
```



