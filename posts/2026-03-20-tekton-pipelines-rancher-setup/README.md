# How to Set Up Tekton Pipelines with Rancher - Pipelines Setup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Tekton, Rancher, Kubernetes, CI/CD, Pipelines, DevOps, Cloud Native

Description: Learn how to install and configure Tekton Pipelines on a Rancher-managed Kubernetes cluster, create reusable tasks, and build end-to-end CI/CD pipelines for container applications.

---

Tekton is a cloud-native CI/CD framework that runs entirely on Kubernetes. Installing it on a Rancher-managed cluster gives you portable, repeatable pipelines that integrate with Rancher's RBAC and project model.

---

## Step 1: Install Tekton Pipelines and Dashboard

```bash
# Install the core Tekton Pipelines components

kubectl apply -f \
  https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Install the Tekton Dashboard for a visual UI
kubectl apply -f \
  https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# Verify pods are running
kubectl get pods -n tekton-pipelines
```

---

## Step 2: Install the Tekton CLI

```bash
# macOS
brew install tektoncd-cli

# Linux (replace VERSION with latest)
curl -LO https://github.com/tektoncd/cli/releases/download/v0.37.0/tkn_0.37.0_Linux_x86_64.tar.gz
tar xvzf tkn_0.37.0_Linux_x86_64.tar.gz tkn
sudo mv tkn /usr/local/bin/
```

---

## Step 3: Create a Task

A Task is the basic unit in Tekton. This task builds and pushes a Docker image using Kaniko (a daemon-less builder):

```yaml
# task-build-push.yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: build-push-image
  namespace: my-app
spec:
  params:
    - name: IMAGE
      description: Full image reference including tag
    - name: CONTEXT
      description: Docker build context path
      default: "."
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:latest
      args:
        # Build from the workspace and push to the registry
        - --dockerfile=Dockerfile
        - --context=dir://$(workspaces.source.path)/$(params.CONTEXT)
        - --destination=$(params.IMAGE)
        - --skip-tls-verify
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      secret:
        secretName: docker-registry-credentials
  workspaces:
    - name: source
```

---

## Step 4: Create a Pipeline

This Pipeline chains a git-clone task with the build-push task:

```yaml
# pipeline-build-deploy.yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-deploy
  namespace: my-app
spec:
  params:
    - name: repo-url
    - name: image-name
    - name: revision
      default: main
  workspaces:
    - name: shared-workspace
  tasks:
    - name: clone
      taskRef:
        name: git-clone
        kind: ClusterTask
      params:
        - name: url
          value: $(params.repo-url)
        - name: revision
          value: $(params.revision)
      workspaces:
        - name: output
          workspace: shared-workspace

    - name: build
      # Run build only after clone finishes
      runAfter: [clone]
      taskRef:
        name: build-push-image
      params:
        - name: IMAGE
          value: $(params.image-name):$(tasks.clone.results.commit)
      workspaces:
        - name: source
          workspace: shared-workspace
```

---

## Step 5: Create a PipelineRun

A PipelineRun triggers the pipeline with concrete parameter values:

```yaml
# pipelinerun-example.yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: build-and-deploy-run-
  namespace: my-app
spec:
  pipelineRef:
    name: build-and-deploy
  params:
    - name: repo-url
      value: https://github.com/my-org/my-app.git
    - name: image-name
      value: registry.example.com/my-org/my-app
  workspaces:
    - name: shared-workspace
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 1Gi
```

```bash
kubectl apply -f pipelinerun-example.yaml
# Watch pipeline progress
tkn pipelinerun logs -f -n my-app
```

---

## Step 6: Set Up Tekton Triggers for Webhooks

Install Tekton Triggers to automatically start pipelines on GitHub pushes:

```bash
kubectl apply -f \
  https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
```

Then create an `EventListener` that accepts GitHub webhook events and starts the pipeline automatically. See the [Tekton Triggers docs](https://tekton.dev/docs/triggers/) for full details.

---

## Best Practices

- Use **ClusterTasks** for common operations like `git-clone` so they are available cluster-wide.
- Store Tekton pipelines in Git and manage them with Rancher Fleet for GitOps.
- Use **PipelineResource** workspaces backed by Longhorn PVCs for large build caches.
