# How to Configure ML Training Pipelines on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, ML Pipelines, Kubeflow, Argo Workflows, Kubernetes, MLOps

Description: Build automated ML training pipelines on Rancher using Argo Workflows or Kubeflow Pipelines with data preparation, training, evaluation, and model registry steps.

## Introduction

ML training pipelines automate the repetitive work of data preparation, model training, evaluation, and deployment. On Rancher, Argo Workflows and Kubeflow Pipelines are the two main options, with Argo being more general-purpose and Kubeflow providing ML-specific abstractions.

## Option 1: Argo Workflows for ML Pipelines

```bash
# Install Argo Workflows
kubectl create namespace argo
kubectl apply -n argo \
  -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.0/install.yaml
```

```yaml
# ml-training-workflow.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: ml-training-pipeline
  namespace: argo
spec:
  entrypoint: train-pipeline
  arguments:
    parameters:
      - name: dataset-version
        value: "v2024-03"
      - name: model-type
        value: "random-forest"

  templates:
    - name: train-pipeline
      dag:
        tasks:
          - name: prepare-data
            template: data-prep
          - name: train-model
            template: trainer
            dependencies: [prepare-data]
            arguments:
              parameters:
                - name: data-path
                  value: "{{tasks.prepare-data.outputs.parameters.data-path}}"
          - name: evaluate
            template: evaluator
            dependencies: [train-model]
          - name: register-model
            template: model-registry
            dependencies: [evaluate]
            when: "{{tasks.evaluate.outputs.parameters.accuracy}} > 0.90"

    - name: data-prep
      container:
        image: myregistry/data-prep:latest
        command: [python, /app/prepare.py]
        args: ["--dataset-version", "{{workflow.parameters.dataset-version}}"]
      outputs:
        parameters:
          - name: data-path
            valueFrom:
              path: /tmp/data-path.txt

    - name: trainer
      container:
        image: myregistry/trainer:latest
        command: [python, /app/train.py]
        args:
          - --data-path
          - "{{inputs.parameters.data-path}}"
          - --model-type
          - "{{workflow.parameters.model-type}}"
        resources:
          requests:
            nvidia.com/gpu: "1"
            memory: "8Gi"

    - name: evaluator
      container:
        image: myregistry/evaluator:latest
        command: [python, /app/evaluate.py]
      outputs:
        parameters:
          - name: accuracy
            valueFrom:
              path: /tmp/accuracy.txt

    - name: model-registry
      container:
        image: myregistry/registry:latest
        command: [python, /app/register.py]
        env:
          - name: MLFLOW_TRACKING_URI
            value: https://mlflow.example.com
```

## Option 2: Kubeflow Pipelines

```python
# kubeflow_pipeline.py
import kfp
from kfp import dsl, compiler

@dsl.component(base_image="python:3.11")
def data_preparation(dataset_path: str, output_path: dsl.Output[dsl.Dataset]):
    """Prepare training data."""
    import pandas as pd
    df = pd.read_csv(dataset_path)
    # Preprocessing logic
    df.to_csv(output_path.path, index=False)

@dsl.component(base_image="python:3.11", packages_to_install=["scikit-learn"])
def train_model(
    dataset: dsl.Input[dsl.Dataset],
    model: dsl.Output[dsl.Model],
    n_estimators: int = 100
) -> float:
    """Train and return accuracy."""
    from sklearn.ensemble import RandomForestClassifier
    from sklearn.model_selection import train_test_split
    import pandas as pd, joblib

    df = pd.read_csv(dataset.path)
    X, y = df.drop("target", axis=1), df["target"]
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

    clf = RandomForestClassifier(n_estimators=n_estimators)
    clf.fit(X_train, y_train)
    accuracy = clf.score(X_test, y_test)
    joblib.dump(clf, model.path)
    return accuracy

@dsl.pipeline(name="ml-training-pipeline")
def pipeline(dataset_path: str = "s3://data/training.csv"):
    prep_task = data_preparation(dataset_path=dataset_path)
    train_task = train_model(dataset=prep_task.outputs["output_path"])

compiler.Compiler().compile(pipeline, "pipeline.yaml")
```

## Conclusion

ML training pipelines on Rancher enable reproducible, automated model development. Argo Workflows is more flexible and supports any containerized workload, while Kubeflow Pipelines provides ML-specific features like artifact tracking and component caching. Choose based on your team's existing tools and ML framework preferences.
