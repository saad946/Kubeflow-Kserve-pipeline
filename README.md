# Kubeflow Pipelines & KServe Setup Guide

Complete setup for Kubeflow Pipelines v2.4.0 and KServe on Kind cluster with S3 model serving.

## Prerequisites

- Docker Desktop, Python 3.8+, pip, 8GB+ RAM
- AWS CLI configured with S3 access

## Quick Setup

### 1. Kind Cluster & Python Environment

```bash
# Install Kind
brew install kind

# Create cluster
kind create cluster --name kind-cluster
kubectl config use-context kind-kind-cluster

# Setup Python environment
python3 -m venv .kfp
source .kfp/bin/activate
pip install kfp
```

### 2. Install Kubeflow Pipelines v2.4.0

```bash
export PIPELINE_VERSION=2.4.0
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"
kubectl wait --for condition=established --timeout=60s crd/applications.app.k8s.io
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/env/platform-agnostic?ref=$PIPELINE_VERSION"
```

### 3. Fix Image Issues

```bash
# Fix minio image
docker pull minio/minio:RELEASE.2019-08-14T20-37-41Z
docker tag minio/minio:RELEASE.2019-08-14T20-37-41Z gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance
kind load docker-image gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance --name kind-cluster
kubectl delete pod -n kubeflow -l app=minio
kubectl delete pod -n kubeflow -l app=ml-pipeline
```

### 4. Install KServe

```bash
curl -s "https://raw.githubusercontent.com/kserve/kserve/release-0.14/hack/quick_install.sh" | bash
```

### 5. Configure AWS & S3 Access

```bash
# Create namespace
kubectl create namespace kserve-test

# Create AWS credentials secret
kubectl create secret generic aws-s3-credentials \
  --from-literal=AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id) \
  --from-literal=AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key) \
  -n kserve-test

# Create ServiceAccount
kubectl create serviceaccount sklearn-iris-sa -n kserve-test
kubectl patch serviceaccount sklearn-iris-sa -n kserve-test -p '{"secrets": [{"name": "aws-s3-credentials"}]}'
```

### 6. Deploy InferenceService

```bash
kubectl apply -n kserve-test -f - <<EOF
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
  namespace: "kserve-test"
  annotations:
    autoscaling.knative.dev/target: "10"
    autoscaling.knative.dev/minScale: "0"
    autoscaling.knative.dev/maxScale: "10"
    serving.kserve.io/enable-prometheus-scraping: "true"
spec:
  predictor:
    canaryTrafficPercent: 0
    model:
      modelFormat:
        name: sklearn
      storageUri: "s3://kubeflow-iris-pipeline/models/iris/model_20251112_201850.joblib"
      resources:
        requests:
          memory: 512Mi
          cpu: 500m
        limits:
          memory: 1Gi
          cpu: 1
    serviceAccountName: sklearn-iris-sa
EOF
```

### 7. Test Inference

```bash
# Create test input
cat > iris-input.json <<EOF
{
  "instances": [
    [6.8, 2.8, 4.8, 1.4],
    [6.0, 3.4, 4.5, 1.6]
  ]
}
EOF

# Test via port-forward
POD_NAME=$(kubectl get pod -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n kserve-test $POD_NAME 8082:8080
curl -X POST http://localhost:8082/v1/models/sklearn-iris:predict -H "Content-Type: application/json" -d @./iris-input.json
```

## Metrics & Observability

### Enable Prometheus

```bash
# Install Prometheus
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false

# Configure PodMonitor
kubectl apply -n kserve-test -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: sklearn-iris-metrics
  namespace: kserve-test
spec:
  selector:
    matchLabels:
      serving.kserve.io/inferenceservice: sklearn-iris
  podMetricsEndpoints:
  - port: user-port
    path: /metrics
    interval: 30s
EOF
```

### Access Metrics

```bash
# Direct pod metrics
POD_NAME=$(kubectl get pod -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n kserve-test $POD_NAME 8083:8080
curl http://localhost:8083/metrics | grep '^request_'

# Prometheus UI
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Open: http://localhost:9090
```

**Key Metrics:**
- `request_preprocess_seconds` - Preprocessing latency
- `request_predict_seconds` - Prediction latency
- `request_postprocess_seconds` - Post-processing latency

**PromQL Queries:**
```bash
# P95 Latency
curl 'http://localhost:9090/api/v1/query?query=histogram_quantile(0.95, rate(request_predict_seconds_bucket[5m]))'

# Average Latency
curl 'http://localhost:9090/api/v1/query?query=rate(request_predict_seconds_sum[5m]) / rate(request_predict_seconds_count[5m])'

# Request Rate
curl 'http://localhost:9090/api/v1/query?query=sum(rate(request_predict_seconds_count[5m]))'
```

## Autoscaling

```bash
# Configure autoscaling
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{
  "metadata": {
    "annotations": {
      "autoscaling.knative.dev/target": "10",
      "autoscaling.knative.dev/minScale": "0",
      "autoscaling.knative.dev/maxScale": "10"
    }
  }
}'

# Monitor
kubectl get podautoscaler -n kserve-test
kubectl get pods -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -w
```

**Parameters:**
- `target`: Concurrent requests per pod (default: 100)
- `minScale`: Minimum pods (0 = scale to zero)
- `maxScale`: Maximum pods

## Canary Rollouts

```bash
# Deploy canary (5% traffic to new model)
kubectl apply -n kserve-test -f - <<EOF
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
  namespace: "kserve-test"
  annotations:
    autoscaling.knative.dev/target: "10"
    autoscaling.knative.dev/minScale: "0"
    autoscaling.knative.dev/maxScale: "10"
    serving.kserve.io/enable-prometheus-scraping: "true"
spec:
  predictor:
    canaryTrafficPercent: 5
    model:
      modelFormat:
        name: sklearn
      storageUri: "s3://kubeflow-iris-pipeline/models/iris/model_20251113_120000.joblib"
      resources:
        requests:
          memory: 512Mi
          cpu: 500m
        limits:
          memory: 1Gi
          cpu: 1
    serviceAccountName: sklearn-iris-sa
EOF

# Monitor traffic split
kubectl get isvc sklearn-iris -n kserve-test -o jsonpath='{.status.components.predictor.traffic[*]}' | python3 -m json.tool

# Increase canary traffic
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{"spec":{"predictor":{"canaryTrafficPercent":25}}}'
```

**Note:** Update `storageUri` with your new model path and adjust `canaryTrafficPercent` as needed.

## Model Explainability (XAI)

```bash
# Deploy with explainer
kubectl apply -n kserve-test -f - <<EOF
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
  namespace: "kserve-test"
  annotations:
    autoscaling.knative.dev/target: "10"
    autoscaling.knative.dev/minScale: "0"
    autoscaling.knative.dev/maxScale: "10"
    serving.kserve.io/enable-prometheus-scraping: "true"
spec:
  predictor:
    canaryTrafficPercent: 0
    model:
      modelFormat:
        name: sklearn
      storageUri: "s3://kubeflow-iris-pipeline/models/iris/model_20251112_201850.joblib"
      resources:
        requests:
          memory: 512Mi
          cpu: 500m
        limits:
          memory: 1Gi
          cpu: 1
    serviceAccountName: sklearn-iris-sa
  explainer:
    containers:
    - name: explainer
      image: kserve/alibi-explainer:latest
      env:
      - name: EXPLAINER_TYPE
        value: "AnchorTabular"
      - name: STORAGE_URI
        value: "s3://kubeflow-iris-pipeline/models/iris/model_20251112_201850.joblib"
      resources:
        requests:
          memory: 256Mi
          cpu: 250m
        limits:
          memory: 512Mi
          cpu: 500m
    serviceAccountName: sklearn-iris-sa
EOF

# Request explanation
EXPLAINER_POD=$(kubectl get pod -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris,serving.kserve.io/component=explainer -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n kserve-test $EXPLAINER_POD 8089:8080
curl -X POST http://localhost:8089/v1/models/sklearn-iris:explain -H "Content-Type: application/json" -d @./iris-input.json
```

## Access Services

```bash
# Kubeflow Pipelines UI
kubectl port-forward -n kubeflow svc/ml-pipeline-ui 8080:80
# Open: http://localhost:8080

# Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Open: http://localhost:9090
```

## Common Issues

1. **Minio Image Error**: Pull `minio/minio:RELEASE.2019-08-14T20-37-41Z` and tag as `gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance`
2. **Memory Issues**: Reduce memory requests (512Mi request, 1Gi limit)
3. **S3 Access**: Use ServiceAccount with AWS credentials secret (not `storageSecret`)
4. **Context**: Always use `kubectl config use-context kind-kind-cluster`

## Files

- `pipeline.py` - ML pipeline definition
- `pipeline.yaml` - Compiled pipeline
- `iris-input.json` - Test input data

**Note:** All KServe YAML configurations are included inline in this README. No separate YAML files needed.

## Quick Reference

```bash
# Context
kubectl config use-context kind-kind-cluster

# Status
kubectl get pods -n kubeflow
kubectl get isvc sklearn-iris -n kserve-test

# Test
POD_NAME=$(kubectl get pod -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n kserve-test $POD_NAME 8082:8080
curl -X POST http://localhost:8082/v1/models/sklearn-iris:predict -H "Content-Type: application/json" -d @./iris-input.json

# Metrics
kubectl port-forward -n kserve-test $POD_NAME 8083:8080
curl http://localhost:8083/metrics | grep '^request_'

# Autoscaling
kubectl get podautoscaler -n kserve-test

# Canary
kubectl get isvc sklearn-iris -n kserve-test -o jsonpath='{.status.components.predictor.traffic[*]}' | python3 -m json.tool
```
