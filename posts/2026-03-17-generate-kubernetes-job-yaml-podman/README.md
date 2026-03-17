# How to Generate a Kubernetes Job YAML with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Kubernetes, YAML, Jobs, Batch

Description: Learn how to generate a Kubernetes Job manifest from a Podman container for batch processing workloads.

---

> Kubernetes Jobs run containers to completion, and Podman can generate the Job YAML from a local container.

A Kubernetes Job creates one or more pods and ensures they run to successful completion. Jobs are used for batch processing, database migrations, data imports, and one-off tasks. Podman can generate a Job manifest from a container you have tested locally.

---

## Running a Batch Container Locally

```bash
# Run a batch processing container
podman run -d --name data-processor \
  -e INPUT_PATH=/data/input \
  -e OUTPUT_PATH=/data/output \
  -v batch-input:/data/input:ro \
  -v batch-output:/data/output \
  docker.io/library/alpine \
  sh -c "echo 'Processing data...' && sleep 5 && echo 'Done' > /data/output/result.txt"
```

## Generating the Job YAML

```bash
# Generate a Kubernetes Job manifest
podman generate kube --type job data-processor > job.yaml

# View the generated file
cat job.yaml
```

## Understanding the Job Structure

```yaml
# Example structure:
# apiVersion: batch/v1
# kind: Job
# metadata:
#   name: data-processor-job
# spec:
#   template:
#     spec:
#       containers:
#         - name: data-processor
#           image: docker.io/library/alpine
#           command:
#             - sh
#             - -c
#             - "echo 'Processing data...' && sleep 5 && echo 'Done' > /data/output/result.txt"
#           env:
#             - name: INPUT_PATH
#               value: /data/input
#           volumeMounts:
#             - mountPath: /data/input
#               name: batch-input
#               readOnly: true
#       restartPolicy: Never
```

## Use Case: Database Migration Job

```bash
# Run a migration container locally
podman run --name db-migrate \
  -e DATABASE_URL=postgresql://admin:secret@localhost:5432/mydb \
  docker.io/library/alpine \
  sh -c "echo 'Running migrations...' && sleep 3 && echo 'Migrations complete'"

# Generate the Job YAML
podman generate kube --type job db-migrate > migration-job.yaml
```

## Use Case: Data Import Job

```bash
# Run a data import container
podman run --name importer \
  -v import-data:/data:ro \
  docker.io/library/alpine \
  sh -c "wc -l /data/*.csv && echo 'Import complete'"

# Generate the Job YAML
podman generate kube --type job importer > import-job.yaml
```

## Customizing the Generated Job

After generating the YAML, you can add Kubernetes-specific fields:

```bash
# Generate the base Job YAML
podman generate kube --type job data-processor > job.yaml

# Edit to add backoff limit and completions
# spec:
#   backoffLimit: 3
#   completions: 1
#   parallelism: 1
#   activeDeadlineSeconds: 300
```

## Applying the Job to Kubernetes

```bash
# Apply the Job to a cluster
kubectl apply -f job.yaml

# Watch the Job progress
kubectl get jobs -w

# View the pod logs
kubectl logs job/data-processor-job

# Clean up completed jobs
kubectl delete job data-processor-job
```

## Replaying the Job with Podman

```bash
# Podman can also play Job YAML locally
podman play kube job.yaml

# This runs the container to completion
podman ps -a --format "{{.Names}} {{.Status}}"
```

## Summary

Use `podman generate kube --type job` to create a Kubernetes Job manifest from a batch container tested locally. The generated YAML captures the container's image, command, environment variables, and volume mounts. Customize with Kubernetes-specific fields like `backoffLimit` and `completions` before deploying.
