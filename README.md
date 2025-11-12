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

### Create InferenceService

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
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
EOF
```

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

**Expected response:** `{"predictions":[1,1]}`

**Note:** Predictions [1,1] mean both iris samples are classified as class 1 (versicolor).

### Issues Solved

1. **Insufficient Memory Error:**
   - **Error:** `0/1 nodes are available: 1 Insufficient memory`
   - **Cause:** Default KServe memory request (2Gi) exceeded available cluster memory
   - **Fix:** If you encounter this issue, add `resources` section to predictor model specification with reduced memory (512Mi request, 1Gi limit)

2. **Pod Stuck in PodInitializing:**
   - **Cause:** Large images (sklearnserver, storage-initializer) taking time to pull
   - **Fix:** Wait for images to finish pulling (normal behavior, takes 2-5 minutes)
   - **Verification:** Check pod events: `kubectl describe pod <pod-name> -n kserve-test`

## 12. Verify Installation

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

## 13. Access Kubeflow Pipelines UI

```bash
# Port forward to access UI
kubectl port-forward -n kubeflow svc/ml-pipeline-ui 8080:80

# Open browser: http://localhost:8080
```

## 14. Troubleshooting

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

6. **Pipeline S3 Bucket Name Error:**
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
```
