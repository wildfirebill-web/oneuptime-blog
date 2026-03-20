# How to Deploy Kubeflow on Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubeflow, MLOps, Kubernetes, Machine-learning

Description: Step-by-step guide to deploying Kubeflow ML platform on Rancher for end-to-end machine learning workflows.

## Introduction

Kubeflow is an ML platform built on Kubernetes that orchestrates ML workflows, notebooks, training, and serving. Deploying it on Rancher gives you a complete MLOps environment with enterprise-grade cluster management.

## Prerequisites

- Rancher cluster with Kubernetes 1.26+
- Minimum 16 GB RAM, 8 CPU per node
- GPU nodes (recommended for training)
- Default StorageClass configured
- kustomize v5.0+ installed

## Step 1: Install kustomize

```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
kustomize version
```

## Step 2: Clone Kubeflow Manifests

```bash
# Clone the Kubeflow manifests repository

git clone https://github.com/kubeflow/manifests.git
cd manifests
git checkout v1.8.0
```

## Step 3: Install Kubeflow Components

```bash
# Install all Kubeflow components
while ! kustomize build example | kubectl apply -f -; do
  echo "Retrying to apply resources..."
  sleep 20
done

# This installs:
# - cert-manager
# - Istio service mesh
# - Knative
# - Kubeflow Pipelines
# - Katib (hyperparameter tuning)
# - Notebook Controller
# - Profile Controller
# - Training Operators (TFJob, PyTorchJob, etc.)
# - KServe (model serving)
# - Kubeflow Dashboard
```

## Step 4: Verify Installation

```bash
# Check all pods are running (may take 5-10 minutes)
kubectl get pods -n kubeflow
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n knative-serving

# Wait for all pods to be ready
kubectl wait --for=condition=Ready pods   --all --all-namespaces   --timeout=600s   --field-selector 'status.phase!=Succeeded'
```

## Step 5: Access Kubeflow Dashboard

```bash
# Port forward the central dashboard
kubectl port-forward svc/istio-ingressgateway   -n istio-system 8080:80

# Default credentials:
# Username: user@example.com
# Password: 12341234
```

## Step 6: Configure Storage for Notebooks

```yaml
# notebook-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jupyter-notebook-storage
  namespace: kubeflow-user-example-com
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: longhorn
```

## Step 7: Create a Kubeflow Pipeline

```python
# pipeline.py - Simple ML pipeline
import kfp
from kfp import dsl

@dsl.component(base_image='python:3.9')
def download_data(url: str, output_path: str):
    import urllib.request
    urllib.request.urlretrieve(url, output_path)

@dsl.component(base_image='python:3.9',
               packages_to_install=['scikit-learn', 'pandas'])
def train_model(data_path: str, model_path: str) -> float:
    import pickle
    import pandas as pd
    from sklearn.ensemble import RandomForestClassifier
    from sklearn.model_selection import train_test_split
    
    df = pd.read_csv(data_path)
    X = df.drop('target', axis=1)
    y = df['target']
    
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
    
    model = RandomForestClassifier(n_estimators=100)
    model.fit(X_train, y_train)
    
    accuracy = model.score(X_test, y_test)
    
    with open(model_path, 'wb') as f:
        pickle.dump(model, f)
    
    return accuracy

@dsl.pipeline(name='ML Training Pipeline')
def ml_pipeline(dataset_url: str = 'https://example.com/data.csv'):
    download_task = download_data(
        url=dataset_url,
        output_path='/tmp/data.csv'
    )
    
    train_task = train_model(
        data_path=download_task.output,
        model_path='/tmp/model.pkl'
    )
    
    print(f"Model accuracy: {train_task.output}")

# Compile and upload
if __name__ == '__main__':
    kfp.compiler.Compiler().compile(
        pipeline_func=ml_pipeline,
        package_path='ml_pipeline.yaml'
    )
```

## Step 8: Configure RBAC for Teams

```yaml
# kubeflow-profile.yaml - Create user profile (namespace)
apiVersion: kubeflow.org/v1
kind: Profile
metadata:
  name: ml-team-profile
spec:
  owner:
    kind: User
    name: mluser@example.com
  resourceQuotaSpec:
    hard:
      cpu: "20"
      memory: 60Gi
      requests.nvidia.com/gpu: "2"
      persistentvolumeclaims: "10"
```

## Conclusion

Kubeflow on Rancher provides a comprehensive MLOps platform covering the complete ML lifecycle from data exploration in notebooks to production model serving. Its integration with Kubernetes means teams get familiar tooling while operations teams maintain control through Rancher's cluster management capabilities.
