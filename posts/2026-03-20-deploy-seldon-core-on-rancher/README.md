# How to Deploy Seldon Core on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Seldon Core, ML Model Serving, Kubernetes, MLOps, Inference

Description: Deploy Seldon Core on Rancher to serve machine learning models at scale with A/B testing, canary deployments, and explainability features.

## Introduction

Seldon Core is an open-source platform for deploying ML models on Kubernetes. It supports scikit-learn, TensorFlow, PyTorch, and custom models via Docker containers. Key features include A/B testing, canary deployments, explainability with SHAP/LIME, and Prometheus metrics out of the box.

## Prerequisites

- Rancher cluster with at least 4 CPUs and 8GB RAM
- Istio or Ambassador installed (for routing)
- `helm` and `kubectl`

## Step 1: Install Seldon Core

```bash
helm repo add seldonio https://storage.googleapis.com/seldon-charts
helm repo update

# Install Seldon Core
helm install seldon-core seldonio/seldon-core-operator \
  --namespace seldon-system \
  --create-namespace \
  --set usageMetrics.enabled=true \
  --set istio.enabled=true    # Set false if not using Istio
```

## Step 2: Verify Installation

```bash
kubectl get pods -n seldon-system
kubectl get crd | grep seldon
```

## Step 3: Deploy a scikit-learn Model

Package your model and deploy it:

```python
# train_and_save.py
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
import joblib

X, y = load_iris(return_X_y=True)
model = RandomForestClassifier()
model.fit(X, y)
joblib.dump(model, 'model.joblib')
```

```yaml
# sklearn-deployment.yaml
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: iris-classifier
  namespace: production
spec:
  predictors:
    - name: default
      replicas: 2
      graph:
        name: classifier
        implementation: SKLEARN_SERVER
        modelUri: s3://my-models/iris-classifier/v1    # Model stored in S3
      componentSpecs:
        - spec:
            containers:
              - name: classifier
                resources:
                  requests:
                    memory: "256Mi"
                    cpu: "100m"
```

```bash
kubectl apply -f sklearn-deployment.yaml
```

## Step 4: Test the Model

```bash
# Get the Seldon service URL
SELDON_URL=$(kubectl get svc istio-ingressgateway -n istio-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Send a prediction request
curl -X POST "http://$SELDON_URL/seldon/production/iris-classifier/api/v1.0/predictions" \
  -H "Content-Type: application/json" \
  -d '{"data": {"ndarray": [[5.1, 3.5, 1.4, 0.2]]}}'
```

## Step 5: A/B Testing with Multiple Models

```yaml
# ab-test-deployment.yaml
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: iris-ab-test
spec:
  predictors:
    - name: model-a
      replicas: 1
      traffic: 70    # 70% of traffic
      graph:
        name: classifier-v1
        implementation: SKLEARN_SERVER
        modelUri: s3://my-models/iris-classifier/v1
    - name: model-b
      replicas: 1
      traffic: 30    # 30% of traffic
      graph:
        name: classifier-v2
        implementation: SKLEARN_SERVER
        modelUri: s3://my-models/iris-classifier/v2
```

## Step 6: Monitor Predictions

```promql
# Prediction rate per model
rate(seldon_api_executor_server_requests_seconds_count{service="iris-classifier"}[5m])

# Model latency p99
histogram_quantile(0.99,
  rate(seldon_api_executor_server_requests_seconds_bucket[5m])
)
```

## Conclusion

Seldon Core on Rancher provides production-grade ML model serving with built-in A/B testing, canary deployments, and observability. The Custom Resource approach integrates naturally with GitOps workflows—model deployments are YAML files that can be version-controlled and deployed through the same pipelines as application code.
