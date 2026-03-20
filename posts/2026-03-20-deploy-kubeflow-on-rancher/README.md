# How to Deploy Kubeflow on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubeflow, Machine Learning, MLOps, Kubernetes, Pipelines

Description: Deploy Kubeflow on Rancher for end-to-end ML platform capabilities including notebooks, pipelines, model serving, and experiment tracking.

## Introduction

Kubeflow is the open-source ML platform for Kubernetes, providing Jupyter Notebooks, ML Pipelines, hyperparameter tuning with Katib, and model serving with KServe. Deploying it on Rancher gives ML teams a managed, scalable platform.

## Prerequisites

- Rancher cluster with 8+ CPUs and 16GB+ RAM
- `kubectl`, `kustomize` v5+
- Persistent storage (Longhorn recommended)

## Step 1: Install kustomize

```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```

## Step 2: Clone Kubeflow Manifests

```bash
export KUBEFLOW_VERSION=v1.8.0
git clone https://github.com/kubeflow/manifests.git
cd manifests
git checkout ${KUBEFLOW_VERSION}
```

## Step 3: Install Kubeflow Components

```bash
# Install all Kubeflow components (this takes 10-20 minutes)

while ! kustomize build example | kubectl apply -f -; do
  echo "Retrying due to resource ordering..."
  sleep 10
done

# Monitor installation
kubectl get pods -n kubeflow --watch
```

## Step 4: Verify Installation

```bash
# Check all pods are running
kubectl get pods -n kubeflow | grep -v Running

# Key components to verify:
# - ml-pipeline (Kubeflow Pipelines)
# - notebook-controller
# - profiles-deployment
# - kserve-controller-manager
```

## Step 5: Access Kubeflow Dashboard

```bash
# Port-forward the Istio ingress gateway
kubectl port-forward svc/istio-ingressgateway \
  -n istio-system 8080:80

# Access at http://localhost:8080
# Default credentials: user@example.com / 12341234
```

## Step 6: Create Your First Notebook

1. Navigate to **Notebooks** in the Kubeflow UI
2. Click **New Notebook**
3. Configure:
   - Name: `ml-training-notebook`
   - Image: `jupyter/tensorflow-notebook:latest`
   - CPU: 2, Memory: 4Gi
4. Click **Launch**

## Step 7: Create an ML Pipeline

```python
# simple_pipeline.py - Kubeflow Pipeline definition
import kfp
from kfp import dsl

@dsl.component
def load_data() -> str:
    """Load training data."""
    return "data_path"

@dsl.component
def train_model(data_path: str) -> str:
    """Train the ML model."""
    return "model_path"

@dsl.pipeline(name="simple-ml-pipeline")
def ml_pipeline():
    load_task = load_data()
    train_task = train_model(data_path=load_task.output)

# Compile and submit
compiler = kfp.compiler.Compiler()
compiler.compile(ml_pipeline, "pipeline.yaml")

client = kfp.Client()
client.create_run_from_pipeline_file("pipeline.yaml")
```

## Conclusion

Kubeflow on Rancher provides a complete MLOps platform for teams building and deploying machine learning models. The pipeline system enables reproducible, automated ML workflows, while the notebook environment gives data scientists familiar tooling. For production use, configure SSO via Rancher's OIDC integration and implement proper resource quotas per team.
