# How to Set Up Tekton Pipelines with Rancher - Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Tekton, Rancher, Kubernetes, CI/CD, Pipeline, GitOps, SUSE Rancher

Description: Learn how to install Tekton Pipelines on a Rancher-managed Kubernetes cluster, create pipeline tasks, and automate container image builds and deployments.

---

Tekton is a Kubernetes-native CI/CD framework that runs pipelines as Kubernetes resources. On Rancher, you can deploy Tekton via Helm and manage pipelines alongside your workloads.

---

## Step 1: Install Tekton Pipelines

```bash
# Install Tekton Pipelines using kubectl

kubectl apply --filename \
  https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

# Install Tekton Triggers (for webhook-triggered pipelines)
kubectl apply --filename \
  https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml

# Install Tekton Dashboard for UI
kubectl apply --filename \
  https://storage.googleapis.com/tekton-releases/dashboard/latest/release.yaml

# Verify all Tekton pods are running
kubectl get pods -n tekton-pipelines
```

---

## Step 2: Install the Tekton CLI

```bash
# Install tkn CLI
curl -LO https://github.com/tektoncd/cli/releases/latest/download/tkn_Linux_x86_64.tar.gz
tar xvzf tkn_Linux_x86_64.tar.gz
sudo mv tkn /usr/local/bin/

# Verify
tkn version
```

---

## Step 3: Create a Pipeline Task

```yaml
# build-task.yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: build-and-push
  namespace: tekton-pipelines
spec:
  params:
    - name: image
      type: string
      description: The image to build and push
    - name: context
      type: string
      default: "."
  workspaces:
    - name: source
      description: The git repository source
  steps:
    - name: build
      image: gcr.io/kaniko-project/executor:latest
      args:
        - "--dockerfile=Dockerfile"
        - "--context=$(workspaces.source.path)/$(params.context)"
        - "--destination=$(params.image)"
        - "--cache=true"
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      secret:
        secretName: registry-credentials
        items:
          - key: .dockerconfigjson
            path: config.json
```

---

## Step 4: Create a Pipeline

```yaml
# pipeline.yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-deploy-pipeline
  namespace: tekton-pipelines
spec:
  params:
    - name: repo-url
      type: string
    - name: image
      type: string
    - name: revision
      type: string
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
      runAfter: [clone]
      taskRef:
        name: build-and-push
      params:
        - name: image
          value: $(params.image)
      workspaces:
        - name: source
          workspace: shared-workspace
```

---

## Step 5: Create a PipelineRun

```yaml
# pipeline-run.yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: build-deploy-run-
  namespace: tekton-pipelines
spec:
  pipelineRef:
    name: build-deploy-pipeline
  params:
    - name: repo-url
      value: https://github.com/my-org/my-app
    - name: image
      value: ghcr.io/my-org/my-app:latest
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
kubectl apply -f pipeline-run.yaml

# Watch the pipeline run
tkn pipelinerun logs -f -n tekton-pipelines
```

---

## Step 6: Set Up a Webhook Trigger

```yaml
# trigger-template.yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: pipeline-trigger-template
  namespace: tekton-pipelines
spec:
  params:
    - name: gitrepourl
    - name: gitrevision
  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: triggered-run-
        namespace: tekton-pipelines
      spec:
        pipelineRef:
          name: build-deploy-pipeline
        params:
          - name: repo-url
            value: $(tt.params.gitrepourl)
          - name: revision
            value: $(tt.params.gitrevision)
          - name: image
            value: ghcr.io/my-org/my-app:$(tt.params.gitrevision)
        workspaces:
          - name: shared-workspace
            volumeClaimTemplate:
              spec:
                accessModes: [ReadWriteOnce]
                resources:
                  requests:
                    storage: 1Gi
```

---

## Best Practices

- Use Tekton ClusterTasks for reusable steps like `git-clone` and `kaniko-build` instead of writing custom tasks for common operations.
- Always use workspaces with `volumeClaimTemplate` for pipeline runs - this ensures each run gets its own ephemeral storage.
- Expose the Tekton Dashboard through an Ingress with authentication to give developers visibility into pipeline runs without `kubectl` access.
