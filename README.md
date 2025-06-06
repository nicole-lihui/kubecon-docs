# LWS Demos

- [LWS Demos](#lws-demos)
  - [Prerequisites](#prerequisites)
    - [Install LWS](#install-lws)
    - [K8s Environment](#k8s-environment)
    - [Download Models](#download-models)
  - [Model Selection](#model-selection)
  - [LWS Multi-Machine Inference Demo](#lws-multi-machine-inference-demo)
  - [HPA](#hpa)
    - [Prerequisites](#prerequisites-1)
    - [ServiceMonitor Exposing vllm Metrics](#servicemonitor-exposing-vllm-metrics)
    - [Create HorizontalPodAutoscaler](#create-horizontalpodautoscaler)
  - [Testing](#testing)
    - [Install vllm Benchmark Demo](#install-vllm-benchmark-demo)
  - [PD Separation Demo](#pd-separation-demo)
    - [Install LMCache](#install-lmcache)
    - [Load PD Proxy Script](#load-pd-proxy-script)
    - [Heterogeneous PD Separation Demo](#heterogeneous-pd-separation-demo)
    - [Homogeneous PD Separation Demo](#homogeneous-pd-separation-demo)

## Prerequisites

### Install LWS

```bash
CHART_VERSION=0.6.1
helm install lws oci://registry.k8s.io/lws/charts/lws \
  --version=$CHART_VERSION \
  --namespace lws-system \
  --create-namespace \
  --wait --timeout 300s
```

### K8s Environment

```bash
kubectl create ns demo-lws
```

Download demo repository
```bash
git clone git@github.com:nicole-lihui/kubecon-docs.git
```

China acceleration
```bash
git clone https://gh-proxy.ygxz.in/https://github.com/nicole-lihui/kubecon-docs
```

switch to default ns `demo-lws`
```bash
kubectl config set-context --current --namespace=demo-lws
```
### Download Models

1. Local download, mount to pod via host path

```bash
pip install modelscope

# Using DeepSeek-R1-Distill-Qwen-1.5B as an example https://modelscope.cn/models/deepseek-ai/DeepSeek-R1-Distill-Qwen-32B
modelscope download --model deepseek-ai/DeepSeek-R1-Distill-Qwen-32B
```

This method is complex in k8s environment, it's recommended to create pods with shared PVC storage

2. Dataset method, mount to PVC via pod (current demo uses this approach)

Using [BaizeAI/dataset](https://github.com/BaizeAI/dataset) project to pre-download models

```bash
kubectl -n demo-lws apply -f ./lws/datasets/deepseek-r1-distill-qwen-32b.yaml
```

## Model Selection

```
MODEL=deepseek-r1-distill-qwen-1-5b
```

## LWS Multi-Machine Inference Demo

[/vllm-workspace/examples/online_serving/multi-node-serving.sh](https://github.com/vllm-project/vllm/blob/main/examples/online_serving/multi-node-serving.sh) is the official vllm multi-machine inference example:

Mainly uses ray for multi-machine inference, the current script adds worker cluster join waiting logic and parameter detection.

**leader run command**

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

**worker run**

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

install

```bash
kubectl -n demo-lws apply -f "./lws/demo/multi-${MODEL}.yaml"
```

uninstall

```bash
kubectl -n demo-lws delete -f "./lws/demo/multi-${MODEL}.yaml"
```

## HPA

### Prerequisites

Install prometheus and prometheus-adapter
You can refer to [vllm-project/production-stack](https://github.com/vllm-project/production-stack/blob/main/observability/README.md)

prometheus-adapter rules custom hpa metrics

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
# Check if hpa metrics are effective
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq | grep vllm_num_requests_waiting -C 10
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/demo-lws/metrics/vllm_num_requests_waiting | jq
```

### ServiceMonitor Exposing vllm Metrics

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

### Create HorizontalPodAutoscaler

```bash
kubectl -n demo-lws apply -f "./lws/hpa/${MODEL}.yaml"
```

uninstall

```bash
kubectl -n demo-lws delete -f "./lws/hpa/${MODEL}.yaml"
```

watch changes

```bash
while true; \
do echo "--------POD----------"; \
kubectl get po; \
echo "--------HPA----------"; \
kubectl get hpa; \
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/demo-lws/metrics/vllm_num_requests_waiting | jq; \
echo "============================="; \
sleep 5; \
done
```

## Testing

### Install vllm Benchmark Demo

```bash
kubectl -n demo-lws apply -f ./lws/demo/vllm-benchmark.yaml

# Enter container
kubectl -n demo-lws exec -it $(kubectl -n demo-lws get pod -l app=vllm-benchmark -o jsonpath='{.items[0].metadata.name}') bash
```

Execute in vllm benchmark container

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
# Download test dataset
git clone https://www.modelscope.cn/datasets/otavia/ShareGPT_Vicuna_unfiltered.git

export MODEL=deepseek-r1-distill-qwen-1-5b
export DATASET_PATH=/vllm-workspace/ShareGPT_Vicuna_unfiltered/ShareGPT_V3_unfiltered_cleaned_split.json

CONCY=150
N_P=1000
```

```bash
python3 /vllm-workspace/benchmarks/benchmark_serving.py \
  --backend vllm \
  --model /data/serving-model \
  --served-model-name "$MODEL" \
  --host "$MODEL" \
  --port 8000 \
  --dataset-name random \
  --random-input 1024 \
  --random-output 514 \
  --random-range-ratio 0.8 \
  --dataset-path "$DATASET_PATH" \
  --max-concurrency $CONCY \
  --num-prompts $N_P
```

## PD Separation Demo

Disaggregated Serving Demo below:

![image](./demo/pd-kvdiff-demo.gif)


### Install LMCache

```bash
kubectl -n demo-lws apply -f ./lws/demo/pd-lmcacheserve.yaml
```

uninstall

```bash
kubectl -n demo-lws delete -f ./lws/demo/pd-lmcacheserve.yaml
```

### Prepare PD Proxy Script

PD separation requires a core router for PD routing, so we use vllm example's python script for demo
[disagg_proxy_server.py](https://github.com/vllm-project/vllm/blob/main/examples/others/lmcache/disagg_prefill_lmcache_v1/disagg_proxy_server.py)

Temporarily mount script via configmap, only for demo

```bash
kubectl -n demo-lws apply -f ./lws/demo/pd-disagg-proxy-server.yaml
```

```bash
kubectl -n demo-lws delete -f ./lws/demo/pd-disagg-proxy-server.yaml
```

### Option 1: Heterogeneous PD Separation Demo

```bash
export  MODEL=deepseek-r1-distill-qwen-1-5b
kubectl -n demo-lws apply -f ./lws/demo/pd-kvdiff-${MODEL}.yaml
```

uninstall

```bash
kubectl -n demo-lws delete -f ./lws/demo/pd-kvdiff-${MODEL}.yaml
```

Testing

```bash
SVC=pd-diff-${MODEL}
PORT=8000
BASE_URL="http://${SVC}:${PORT}"

curl --location "${BASE_URL}/v1/completions" \
  --header 'Content-Type: application/json' \
  --data "$(cat <<EOF
{
  "model": "${MODEL}",
  "prompt": "hi, what is LeaderWorkerSet?"
}
EOF
)"
```

### Option 2: Homogeneous PD Separation Demo

```bash
export  MODEL=deepseek-r1-distill-qwen-1-5b
kubectl -n demo-lws apply -f ./lws/demo/pd-kvboth-${MODEL}.yaml
```


```bash
SVC=pd-both-${MODEL}
PORT=8000
BASE_URL="http://${SVC}:${PORT}"

curl --location "${BASE_URL}/v1/completions" \
  --header 'Content-Type: application/json' \
  --data "$(cat <<EOF
{
  "model": "${MODEL}",
  "prompt": "hi, How do you think about LWS(LeaderWorkerSet)?"
}
EOF
)"
```
