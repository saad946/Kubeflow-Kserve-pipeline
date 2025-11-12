# Kubeflow Pipelines Setup Guide

This guide shows how to set up Kubeflow Pipelines v2.4.0 on a local Kind cluster.

## Prerequisites

- Docker Desktop
- macOS/Linux terminal
- Python 3.8+
- pip
- 8GB+ RAM recommended

## 1. Verify Python and pip Installation

```bash
python3 --version
pip3 --version
```

## 2. Install Kind (Kubernetes in Docker)

```bash
brew install kind
kind version
```

## 3. Create Kind Cluster

```bash
kind create cluster --name kind-cluster
kubectl cluster-info --context kind-kind-cluster
kubectl get nodes
```

**Important:** Make sure you're using the correct kubectl context:
```bash
kubectl config current-context
kubectl config use-context kind-kind-cluster
```

## 4. Setup Python Environment

```bash
# Navigate to project directory
cd /Users/saadullah/Documents/learning/KubFlow-Pipeline-Demo

# Create virtual environment
python3 -m venv .kfp

# Verify virtual environment Python version
.kfp/bin/python --version

# Verify virtual environment pip version
.kfp/bin/pip --version

# Activate virtual environment and check for kfp (should show "kfp not found")
source .kfp/bin/activate
which kfp

# Install KFP SDK
.kfp/bin/pip install kfp

# Verify installation
source .kfp/bin/activate
which kfp
kfp --version
```

## 5. Install Kubeflow Pipelines v2.4.0

```bash
# Set pipeline version
export PIPELINE_VERSION=2.4.0

# Apply cluster-scoped resources (CRDs, namespace, service accounts, cluster roles)
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"

# Wait for CRD to be established
kubectl wait --for condition=established --timeout=60s crd/applications.app.k8s.io

# Apply platform-agnostic environment manifests (deployments, services, configmaps, secrets, PVCs)
kubectl apply -k "github.com/kubeflow/pipelines/manifests/kustomize/env/platform-agnostic?ref=$PIPELINE_VERSION"

# Check pod status
kubectl get pods -n kubeflow
```

## 6. Fix Image Pulling Issues

Some images may fail to pull. Pull them manually and load into Kind cluster:

```bash
# Pull all required images
docker pull ghcr.io/kubeflow/kfp-cache-deployer:2.4.0
docker pull ghcr.io/kubeflow/kfp-cache-server:2.4.0
docker pull ghcr.io/kubeflow/kfp-metadata-envoy:2.4.0
docker pull gcr.io/tfx-oss-public/ml_metadata_store_server:1.14.0
docker pull ghcr.io/kubeflow/kfp-metadata-writer:2.4.0
docker pull ghcr.io/kubeflow/kfp-api-server:2.4.0
docker pull ghcr.io/kubeflow/kfp-persistence-agent:2.4.0
docker pull ghcr.io/kubeflow/kfp-scheduled-workflow-controller:2.4.0
docker pull ghcr.io/kubeflow/kfp-frontend:2.4.0
docker pull ghcr.io/kubeflow/kfp-viewer-crd-controller:2.4.0
docker pull ghcr.io/kubeflow/kfp-visualization-server:2.4.0
docker pull gcr.io/ml-pipeline/mysql:8.0.26
docker pull gcr.io/ml-pipeline/workflow-controller:v3.4.17-license-compliance

# Fix minio image (original image doesn't exist, use alternative)
docker pull minio/minio:RELEASE.2019-08-14T20-37-41Z
docker tag minio/minio:RELEASE.2019-08-14T20-37-41Z gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance

# Load minio image into Kind cluster
kind load docker-image gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance --name kind-cluster

# Restart minio pod to use the loaded image
kubectl delete pod -n kubeflow -l app=minio

# Restart ml-pipeline pod (it was failing because minio wasn't running)
kubectl delete pod -n kubeflow -l app=ml-pipeline

# Verify all pods are running
kubectl get pods -n kubeflow
```

## 7. Install and Verify Istio, Cert Manager, Knative & KServe

```bash
# Install KServe (includes Istio, Cert Manager, and Knative)
curl -s "https://raw.githubusercontent.com/kserve/kserve/release-0.14/hack/quick_install.sh" | bash

# Verify KServe pods
kubectl get pods -n kserve

# Verify Istio pods
kubectl get pods -n istio-system

# Verify Knative Serving pods
kubectl get pods -n knative-serving
```

## 8. Install AWS CLI

```bash
# Download AWS CLI installer
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"

# Install AWS CLI
sudo installer -pkg AWSCLIV2.pkg -target /

# Verify installation
which aws
aws --version
```

## 9. Configure AWS CLI

```bash
# Configure AWS CLI with your credentials
aws configure

# You will be prompted for:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region name (e.g., us-east-1)
# - Default output format (e.g., json)

# Verify configuration
aws configure list
```

**Note:** Create an AWS IAM user with S3 full permissions before configuring AWS CLI.

## 10. Setup KServe InferenceService

### Set Istio Ingress Gateway Environment Variables

```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
```

### Create "kserve-test" Namespace

```bash
kubectl create namespace kserve-test
```

### Create AWS Credentials Secret (for S3 Access)

Create a Kubernetes Secret with AWS credentials for S3 access:

```bash
# Create secret with AWS credentials from AWS CLI config
kubectl create secret generic aws-s3-credentials \
  --from-literal=AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id) \
  --from-literal=AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key) \
  -n kserve-test

# Or create manually with your credentials
kubectl create secret generic aws-s3-credentials \
  --from-literal=AWS_ACCESS_KEY_ID='your-access-key-id' \
  --from-literal=AWS_SECRET_ACCESS_KEY='your-secret-access-key' \
  -n kserve-test
```

### Create ServiceAccount for S3 Access

Create a ServiceAccount that references the AWS credentials secret:

```bash
kubectl create serviceaccount sklearn-iris-sa -n kserve-test

kubectl patch serviceaccount sklearn-iris-sa -n kserve-test -p '{"imagePullSecrets": [], "secrets": [{"name": "aws-s3-credentials"}]}'
```

### Create InferenceService with S3 Storage

Replace `s3://kubeflow-iris-pipeline/models/iris/model_20251112_201850.joblib` with your actual S3 model path:

```bash
kubectl apply -n kserve-test -f - <<EOF
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
spec:
  predictor:
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

**Important Notes for S3:**
- Use `storageUri` with `s3://` prefix pointing to your model file
- Use `serviceAccountName` to reference the ServiceAccount that has access to AWS credentials
- The ServiceAccount must reference the `aws-s3-credentials` secret
- **Model File:** KServe sklearn server can load the model file directly (e.g., `model_20251112_201850.joblib`)
- Ensure your S3 bucket and model file are accessible with the provided AWS credentials

### Check the InferenceService (May show READY: Unknown or False)

```bash
kubectl get inferenceservices sklearn-iris -n kserve-test
```

### Check Istio Ingress Gateway

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

### InferenceService should be Ready Now (READY: True)

```bash
kubectl get isvc sklearn-iris -n kserve-test
```

## 11. Test InferenceService

### Create a Datafile for Inference: iris-input.json

```bash
cat <<EOF > "./iris-input.json"
{
  "instances": [
    [6.8,  2.8,  4.8,  1.4],
    [6.0,  3.4,  4.5,  1.6]
  ]
}
EOF
```

### Set the URL of the Model in the InferenceService as an Environment Variable SERVICE_HOST & Use CURL Command to draw inference from the ML Model to get Prediction

```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice sklearn-iris -n kserve-test -o jsonpath='{.status.url}' | cut -d "/" -f 3)

curl -v -H "Host: ${SERVICE_HOSTNAME}" -H "Content-Type: application/json" "http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/sklearn-iris:predict" -d @./iris-input.json
```

**Expected response:** `{"predictions":[2,2]}` (or similar predictions based on your model)

**Note:** The predictions depend on your trained model. For example, [2,2] means both iris samples are classified as class 2 (virginica).

## 12. KServe Native Metrics & Observability

KServe provides native metrics for monitoring model serving performance. These metrics track latency at each stage of the inference pipeline.

### KServe Native Metrics

KServe exposes the following native metrics (histogram format):

- **`request_preprocess_seconds`** - Preprocessing latency
- **`request_predict_seconds`** - Prediction latency (core model inference)
- **`request_postprocess_seconds`** - Post-processing latency
- **`request_explain_seconds`** - Explainer latency (if explainer enabled)

Each metric provides:
- `_bucket` - Histogram buckets for percentiles (P50, P95, P99)
- `_count` - Total number of requests
- `_sum` - Sum of all request latencies
- `_created` - Timestamp when metric was created

### Enable Prometheus Scraping

Enable Prometheus metrics scraping on the InferenceService:

```bash
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{"metadata":{"annotations":{"serving.kserve.io/enable-prometheus-scraping":"true"}}}'
```

### Install Prometheus

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false

# Wait for Prometheus to be ready
kubectl get pods -n monitoring -w
```

### Configure PodMonitor for KServe Metrics

Create a PodMonitor to scrape KServe metrics:

```bash
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
  - port: http-autometric
    path: /metrics
    interval: 30s
EOF
```

### Access Metrics Directly from Pod

Port-forward to the pod and access metrics endpoint:

```bash
# Get pod name
POD_NAME=$(kubectl get pod -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -o jsonpath='{.items[0].metadata.name}')

# Port-forward to pod (port 8080 has metrics)
kubectl port-forward -n kserve-test $POD_NAME 8083:8080

# In another terminal, view all KServe native metrics
curl http://localhost:8083/metrics | grep '^request_' | grep -E '(preprocess|predict|postprocess|explain)_seconds'

# View specific metric counts and sums
curl http://localhost:8083/metrics | grep -E '^request_(preprocess|predict|postprocess)_seconds_(count|sum)'
```

**Example output:**
```
request_preprocess_seconds_count{model_name="sklearn-iris"} 17.0
request_preprocess_seconds_sum{model_name="sklearn-iris"} 0.0014937499799998477
request_predict_seconds_count{model_name="sklearn-iris"} 17.0
request_predict_seconds_sum{model_name="sklearn-iris"} 0.051023873005760834
request_postprocess_seconds_count{model_name="sklearn-iris"} 17.0
request_postprocess_seconds_sum{model_name="sklearn-iris"} 0.0001197059900732711
```

### Access Prometheus UI

```bash
# Port-forward to Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Open browser: http://localhost:9090
```

### Query KServe Metrics in Prometheus

Use these PromQL queries to analyze KServe metrics:

**1. P95 Prediction Latency:**
```bash
curl 'http://localhost:9090/api/v1/query?query=histogram_quantile(0.95, rate(request_predict_seconds_bucket[5m]))'
```

**2. Average Prediction Latency:**
```bash
curl 'http://localhost:9090/api/v1/query?query=rate(request_predict_seconds_sum[5m]) / rate(request_predict_seconds_count[5m])'
```

**3. Average Preprocessing Latency:**
```bash
curl 'http://localhost:9090/api/v1/query?query=rate(request_preprocess_seconds_sum[5m]) / rate(request_preprocess_seconds_count[5m])'
```

**4. Average Postprocessing Latency:**
```bash
curl 'http://localhost:9090/api/v1/query?query=rate(request_postprocess_seconds_sum[5m]) / rate(request_postprocess_seconds_count[5m])'
```

**5. Request Rate (requests per second):**
```bash
curl 'http://localhost:9090/api/v1/query?query=sum(rate(request_predict_seconds_count[5m]))'
```

**6. Total Request Count:**
```bash
curl 'http://localhost:9090/api/v1/query?query=sum(request_predict_seconds_count)'
```

**7. P95 Preprocessing Latency:**
```bash
curl 'http://localhost:9090/api/v1/query?query=histogram_quantile(0.95, rate(request_preprocess_seconds_bucket[5m]))'
```

**8. P95 Postprocessing Latency:**
```bash
curl 'http://localhost:9090/api/v1/query?query=histogram_quantile(0.95, rate(request_postprocess_seconds_bucket[5m]))'
```

### Key Calculations

- **Average Latency** = `sum / count`
- **P95 Latency** = `histogram_quantile(0.95, rate(bucket[5m]))`
- **Request Rate** = `rate(count[5m])`
- **Total Requests** = `count`

### View Pod Logs (Real-time)

```bash
# View recent logs with timing metrics
POD_NAME=$(kubectl get pod -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n kserve-test $POD_NAME -c kserve-container --tail=50

# Watch logs in real-time
kubectl logs -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -c kserve-container -f
```

**Logs show:**
- `preprocess_ms`: Preprocessing time
- `predict_ms`: Prediction time  
- `postprocess_ms`: Post-processing time
- `http_status`: HTTP response status
- `time:wall` and `time:cpu`: Wall clock and CPU time

## 13. KServe Autoscaling (Handle High Demand)

KServe uses Knative autoscaling to automatically scale model pods based on incoming traffic. This enables:
- **Scale-up**: Automatically add pods when traffic increases
- **Scale-to-zero**: Shut down pods when idle to save costs
- **Latency-based scaling**: Scale based on request latency and concurrent requests

### Configure Autoscaling

Add autoscaling annotations to your InferenceService:

```bash
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{
  "metadata": {
    "annotations": {
      "autoscaling.knative.dev/target": "10",
      "autoscaling.knative.dev/minScale": "0",
      "autoscaling.knative.dev/maxScale": "10"
    }
  }
}'
```

**Autoscaling Parameters:**
- **`autoscaling.knative.dev/target`**: Target number of concurrent requests per pod (default: 100)
  - Lower value = more pods for same traffic (better latency, higher cost)
  - Higher value = fewer pods (lower cost, potentially higher latency)
- **`autoscaling.knative.dev/minScale`**: Minimum number of pods (0 = scale to zero)
- **`autoscaling.knative.dev/maxScale`**: Maximum number of pods (prevents runaway scaling)

### Alternative: Configure in YAML

You can also configure autoscaling when creating the InferenceService:

```bash
kubectl apply -n kserve-test -f - <<EOF
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
  annotations:
    autoscaling.knative.dev/target: "10"
    autoscaling.knative.dev/minScale: "0"
    autoscaling.knative.dev/maxScale: "10"
    serving.kserve.io/enable-prometheus-scraping: "true"
spec:
  predictor:
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

### Monitor Autoscaling

**View Autoscaler Status:**
```bash
# List all autoscalers
kubectl get podautoscaler -n kserve-test

# View autoscaler details
kubectl get podautoscaler sklearn-iris-predictor-00001 -n kserve-test -o yaml

# Watch autoscaler in real-time
kubectl get podautoscaler -n kserve-test -w
```

**View Pod Scaling:**
```bash
# Watch pods scale up/down
kubectl get pods -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -w

# View current pod count
kubectl get pods -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris
```

**View Revision Status:**
```bash
# List revisions
kubectl get revision -n kserve-test

# View revision details
kubectl get revision sklearn-iris-predictor-00001 -n kserve-test -o yaml
```

### Test Autoscaling

**1. Generate Load to Test Scale-Up:**
```bash
# Port-forward to service
POD_NAME=$(kubectl get pod -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n kserve-test $POD_NAME 8084:8080

# In another terminal, generate concurrent requests
for i in {1..100}; do
  curl -s -X POST http://localhost:8084/v1/models/sklearn-iris:predict \
    -H "Content-Type: application/json" \
    -d @./iris-input.json > /dev/null &
done
wait

# Check if pods scaled up
kubectl get pods -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris
kubectl get podautoscaler -n kserve-test
```

**2. Test Scale-to-Zero:**
```bash
# Wait for traffic to stop (typically 60-90 seconds)
# Watch pods scale to zero
kubectl get pods -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -w

# After pods scale to zero, make a request to scale from zero
kubectl port-forward -n kserve-test svc/sklearn-iris-predictor 8085:80
curl -X POST http://localhost:8085/v1/models/sklearn-iris:predict \
  -H "Content-Type: application/json" \
  -d @./iris-input.json

# Check pods scaled from zero
kubectl get pods -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris
```

### Autoscaling Behavior

- **Scale-up**: When concurrent requests exceed `target * current_pods`, new pods are created
- **Scale-down**: When traffic decreases, pods are gradually removed (respects `minScale`)
- **Scale-to-zero**: If `minScale=0` and no traffic for ~60-90 seconds, pods are terminated
- **Scale-from-zero**: First request after scale-to-zero may have cold start latency (~2-5 seconds)

### Production Recommendations

**For Production Workloads:**
```yaml
annotations:
  autoscaling.knative.dev/target: "50"      # 50 concurrent requests per pod
  autoscaling.knative.dev/minScale: "1"     # Always keep 1 pod running (no cold starts)
  autoscaling.knative.dev/maxScale: "100"   # Allow up to 100 pods
```

**For Cost-Optimized Workloads:**
```yaml
annotations:
  autoscaling.knative.dev/target: "10"      # 10 concurrent requests per pod
  autoscaling.knative.dev/minScale: "0"     # Scale to zero when idle
  autoscaling.knative.dev/maxScale: "50"    # Limit maximum pods
```

**For Low-Latency Workloads:**
```yaml
annotations:
  autoscaling.knative.dev/target: "5"      # Fewer concurrent requests = more pods = lower latency
  autoscaling.knative.dev/minScale: "2"     # Keep 2 pods warm
  autoscaling.knative.dev/maxScale: "200"   # Allow aggressive scaling
```

## 14. KServe Canary Rollouts (Advanced Deployment)

Canary deployments allow you to gradually roll out new model versions while monitoring their performance. This minimizes risk by sending a small percentage of traffic to the new model first.

### How Canary Rollouts Work

1. **Initial Deployment**: Deploy your stable model (100% traffic)
2. **Canary Deployment**: Deploy new model version with small traffic percentage (e.g., 5%)
3. **Monitor Performance**: Compare metrics between stable and canary models
4. **Gradual Rollout**: Increase canary traffic (10% → 25% → 50% → 100%)
5. **Rollback**: If issues detected, set canary traffic to 0%

### Step 1: Deploy Initial Model (Stable)

```bash
kubectl apply -n kserve-test -f sklearn-iris-inferenceservice.yaml
```

**File: `sklearn-iris-inferenceservice.yaml`**
```yaml
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
```

### Step 2: Deploy Canary with New Model Version

**Upload your new model to S3 first:**
```bash
# Example: Upload new model version
aws s3 cp new_model.joblib s3://kubeflow-iris-pipeline/models/iris/model_20251113_120000.joblib
```

**Update InferenceService with canary traffic:**
```bash
kubectl apply -n kserve-test -f sklearn-iris-canary.yaml
```

**File: `sklearn-iris-canary.yaml`**
```yaml
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
    canaryTrafficPercent: 5  # 5% traffic to canary, 95% to stable
    model:
      modelFormat:
        name: sklearn
      # Replace with your new model path
      storageUri: "s3://kubeflow-iris-pipeline/models/iris/model_20251113_120000.joblib"
      resources:
        requests:
          memory: 512Mi
          cpu: 500m
        limits:
          memory: 1Gi
          cpu: 1
    serviceAccountName: sklearn-iris-sa
```

### Step 3: Monitor Traffic Distribution

```bash
# View traffic split
kubectl get isvc sklearn-iris -n kserve-test -o jsonpath='{.status.components.predictor.traffic[*]}' | python3 -m json.tool

# View revisions
kubectl get revision -n kserve-test

# View detailed traffic distribution
kubectl get isvc sklearn-iris -n kserve-test -o yaml | grep -A 10 "traffic:"
```

**Expected output:**
```json
[
  {
    "latestRevision": true,
    "percent": 5,
    "revisionName": "sklearn-iris-predictor-00002"
  },
  {
    "latestRevision": false,
    "percent": 95,
    "revisionName": "sklearn-iris-predictor-00001",
    "tag": "prev"
  }
]
```

### Step 4: Monitor Canary Performance in Prometheus

Both models will have separate `model_name` labels in Prometheus metrics:

```bash
# Compare prediction latency between stable and canary
# Stable model metrics
curl 'http://localhost:9090/api/v1/query?query=rate(request_predict_seconds_sum{model_name="sklearn-iris"}[5m]) / rate(request_predict_seconds_count{model_name="sklearn-iris"}[5m])'

# Canary model metrics (will have different revision label)
curl 'http://localhost:9090/api/v1/query?query=rate(request_predict_seconds_sum[5m]) / rate(request_predict_seconds_count[5m])'
```

**View metrics from both revisions:**
```bash
# Port-forward to stable revision pod
POD_STABLE=$(kubectl get pod -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris,serving.kserve.io/revision=sklearn-iris-predictor-00001 -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n kserve-test $POD_STABLE 8087:8080

# Port-forward to canary revision pod
POD_CANARY=$(kubectl get pod -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris,serving.kserve.io/revision=sklearn-iris-predictor-00002 -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n kserve-test $POD_CANARY 8088:8080

# Compare metrics
curl http://localhost:8087/metrics | grep request_predict_seconds_count
curl http://localhost:8088/metrics | grep request_predict_seconds_count
```

### Step 5: Gradually Increase Canary Traffic

```bash
# Increase to 10%
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{"spec":{"predictor":{"canaryTrafficPercent":10}}}'

# Increase to 25%
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{"spec":{"predictor":{"canaryTrafficPercent":25}}}'

# Increase to 50%
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{"spec":{"predictor":{"canaryTrafficPercent":50}}}'

# Increase to 100% (full rollout)
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{"spec":{"predictor":{"canaryTrafficPercent":100}}}'
```

### Step 6: Rollback if Issues Detected

```bash
# Rollback to 0% (all traffic back to stable)
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{"spec":{"predictor":{"canaryTrafficPercent":0}}}'

# Or update to use stable model storageUri
kubectl apply -n kserve-test -f sklearn-iris-inferenceservice.yaml
```

### Canary Deployment Best Practices

1. **Start Small**: Begin with 5% traffic to canary
2. **Monitor Metrics**: Watch latency, error rates, and prediction accuracy
3. **Wait Period**: Monitor for at least 15-30 minutes before increasing traffic
4. **Gradual Increase**: Increase traffic in steps (5% → 10% → 25% → 50% → 100%)
5. **Set Alerts**: Configure Prometheus alerts for error rate spikes
6. **Rollback Plan**: Always have a rollback strategy ready

### Testing Canary Deployment

```bash
# Generate traffic to test both models
for i in {1..100}; do
  curl -s -X POST http://localhost:8086/v1/models/sklearn-iris:predict \
    -H "Content-Type: application/json" \
    -d @./iris-input.json > /dev/null &
done
wait

# Check which revision handled each request (check logs)
kubectl logs -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -c kserve-container | tail -20
```

## 15. Verify Installation

```bash
# Check all pods are running
kubectl get pods -n kubeflow

# Check for any errors
kubectl get pods -n kubeflow | grep -E "Error|CrashLoopBackOff|ImagePullBackOff"

# View ml-pipeline logs
kubectl logs -n kubeflow -l app=ml-pipeline --tail=50

# View minio logs
kubectl logs -n kubeflow -l app=minio --tail=50
```

## 14. Access Kubeflow Pipelines UI

```bash
# Port forward to access UI
kubectl port-forward -n kubeflow svc/ml-pipeline-ui 8080:80

# Open browser: http://localhost:8080
```

## 15. Troubleshooting

### Check kubectl context
```bash
kubectl config current-context
kubectl config use-context kind-kind-cluster
```

### Restart Kind cluster
```bash
# Quick restart
docker restart kind-cluster-control-plane

# Clean restart (delete and recreate)
kind delete cluster --name kind-cluster
kind create cluster --name kind-cluster
kubectl config use-context kind-kind-cluster
```

### Common Issues Fixed

1. **Minio Image Pull Error:**
   - Error: `gcr.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance: not found`
   - Fix: Pull `minio/minio:RELEASE.2019-08-14T20-37-41Z` and tag it as the expected image

2. **ml-pipeline CrashLoopBackOff:**
   - Error: `Failed to check if object store bucket exists. Error: dial tcp 10.96.152.66:9000: connect: connection refused`
   - Fix: Restart minio pod first, then restart ml-pipeline pod

3. **Image Pull Issues:**
   - Pull images manually and load into Kind cluster using `kind load docker-image`

4. **KServe InferenceService Memory Issue:**
   - **Error:** `0/1 nodes are available: 1 Insufficient memory`
   - **Fix:** Reduce memory requests in InferenceService spec (512Mi request, 1Gi limit)

5. **KServe Pod Stuck in PodInitializing:**
   - **Cause:** Large container images being pulled (normal, takes 2-5 minutes)
   - **Fix:** Wait for image pulls to complete, check pod events for progress

6. **KServe S3 Access Issues:**
   - **Error:** `NoCredentialsError: Unable to locate credentials` in storage-initializer
   - **Cause:** Storage-initializer init container cannot access AWS credentials
   - **Fix:** 
     - Create a ServiceAccount that references the AWS credentials secret
     - Use `serviceAccountName` in the InferenceService spec (not `storageSecret` which doesn't exist)
     - The ServiceAccount allows the storage-initializer to access the credentials

7. **Pipeline S3 Bucket Name Error:**
   - **Error:** `Invalid bucket name " kubeflow-iris-pipeline": Bucket name must match the regex`
   - **Cause:** Leading/trailing whitespace in bucket name parameter
   - **Fix:** Added `.strip()` to remove whitespace from `s3_bucket` and `s3_key` in pipeline code

### Check Pod Logs
```bash
# Check specific pod logs
kubectl logs -n kubeflow <pod-name> --tail=50

# Check for errors
kubectl logs -n kubeflow <pod-name> 2>&1 | grep -iE "error|fatal|panic"
```

## Files

- `.kfp/` - Python virtual environment with kfp SDK
- `pipeline.py` - ML pipeline definition (Python DSL)
- `pipeline.yaml` - Compiled pipeline YAML
- `iris-input.json` - Test input file for InferenceService
- `test-inference.sh` - Script to test KServe InferenceService
- `sklearn-iris-inferenceservice.yaml` - Stable InferenceService configuration (0% canary)
- `sklearn-iris-canary.yaml` - Canary deployment configuration (5% traffic to new model)
- `README.md` - This guide

## Quick Reference

```bash
# Activate virtual environment
source .kfp/bin/activate

# Check kfp version
kfp --version

# Switch to Kind cluster context
kubectl config use-context kind-kind-cluster

# Check all pods status
kubectl get pods -n kubeflow

# Verify KServe, Istio, and Knative
kubectl get pods -n kserve
kubectl get pods -n istio-system
kubectl get pods -n knative-serving

# Check AWS CLI
aws --version
aws configure list

# Set Istio Ingress Gateway variables
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')

# Test InferenceService
SERVICE_HOSTNAME=$(kubectl get inferenceservice sklearn-iris -n kserve-test -o jsonpath='{.status.url}' | cut -d "/" -f 3)
curl -v -H "Host: ${SERVICE_HOSTNAME}" -H "Content-Type: application/json" "http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/sklearn-iris:predict" -d @./iris-input.json

# View KServe Native Metrics
POD_NAME=$(kubectl get pod -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n kserve-test $POD_NAME 8083:8080  # Access metrics endpoint
curl http://localhost:8083/metrics | grep '^request_'  # View KServe native metrics

# Access Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Query: histogram_quantile(0.95, rate(request_predict_seconds_bucket[5m]))

# Configure Autoscaling
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{"metadata":{"annotations":{"autoscaling.knative.dev/target":"10","autoscaling.knative.dev/minScale":"0","autoscaling.knative.dev/maxScale":"10"}}}'

# Monitor Autoscaling
kubectl get podautoscaler -n kserve-test  # View autoscaler status
kubectl get pods -n kserve-test -l serving.kserve.io/inferenceservice=sklearn-iris -w  # Watch pods scale

# Deploy Canary (5% traffic to new model)
kubectl apply -n kserve-test -f sklearn-iris-canary.yaml

# Monitor Canary Traffic Split
kubectl get isvc sklearn-iris -n kserve-test -o jsonpath='{.status.components.predictor.traffic[*]}' | python3 -m json.tool

# Increase Canary Traffic
kubectl patch inferenceservice sklearn-iris -n kserve-test --type merge -p '{"spec":{"predictor":{"canaryTrafficPercent":25}}}'
```
