# How to Configure Per-Cluster Registry Access in Portainer for Kubernetes (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Registry, DevOps

Description: Learn how to configure registry access on a per-Kubernetes-cluster basis in Portainer for secure multi-cluster image management.

## Introduction

In multi-cluster Portainer environments, different Kubernetes clusters may use different container registries. Portainer allows you to configure registry access per cluster - ensuring each cluster's deployments can only use approved registries with appropriate credentials. This guide covers configuring per-cluster registry access in Portainer for Kubernetes.

## Prerequisites

- Portainer BE (per-cluster registry access is a BE feature)
- Multiple Kubernetes environments connected to Portainer
- Container registries configured in Portainer

## Step 1: Configure Global Registries

First, add all registries to the global Portainer registry list:

1. Go to **Registries** in Portainer
2. Add your registries (Docker Hub, ECR, ACR, private registry, etc.)
3. These are now available globally

## Step 2: Configure Cluster-Specific Registry Access

For each Kubernetes cluster, specify which registries it can access:

1. Go to **Settings → Environments** (or **Home → click environment**)
2. Click on a Kubernetes environment
3. Navigate to **Registry access**
4. Select which registries this cluster can use:

```bash
Environment: production-k8s
Allowed registries:
  [x] production-registry.company.com    (allow)
  [x] Docker Hub (authenticated)          (allow)
  [ ] staging-registry.company.com       (deny)
  [ ] dev-registry.company.com           (deny)
```

## Step 3: Create Kubernetes Image Pull Secrets

For Kubernetes to pull from private registries, create image pull secrets in each namespace:

```bash
# Create pull secret for a private registry

kubectl create secret docker-registry regcred \
  --docker-server=registry.company.com \
  --docker-username=portainer-user \
  --docker-password=password \
  --docker-email=devops@company.com \
  --namespace=production

# Verify
kubectl get secret regcred -n production
```

Portainer can create these secrets automatically when you configure registry access for a cluster.

## Step 4: Configure Default Service Account Pull Secrets

To automatically use the pull secret for all pods in a namespace:

```bash
# Patch the default service account to include the pull secret
kubectl patch serviceaccount default \
  -n production \
  -p '{"imagePullSecrets":[{"name":"regcred"}]}'
```

Or configure it in your deployment YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  template:
    spec:
      imagePullSecrets:
        - name: regcred   # Reference the pull secret
      containers:
        - name: app
          image: registry.company.com/myapp:latest
```

## Step 5: Portainer Kubernetes Registry Configuration via Manifest

Configure registry credentials via Kubernetes secrets managed through Portainer's manifest editor:

```yaml
# registry-secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-dockerconfig>
```

To generate the base64-encoded config:

```bash
# Create the Docker config JSON
cat > /tmp/dockerconfig.json << EOF
{
  "auths": {
    "registry.company.com": {
      "username": "portainer-user",
      "password": "mypassword",
      "auth": "$(echo -n 'portainer-user:mypassword' | base64)"
    }
  }
}
EOF

# Base64 encode it
base64 -w0 /tmp/dockerconfig.json
```

## Step 6: Use ECR with Kubernetes

ECR requires a token refresh mechanism. Use the `aws-ecr-credential-helper` or a dedicated operator:

```yaml
# ecr-credentials-updater CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ecr-credentials-updater
  namespace: default
spec:
  schedule: "0 */6 * * *"    # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ecr-updater
          containers:
            - name: ecr-updater
              image: odaniait/aws-kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  TOKEN=$(aws ecr get-login-password --region us-east-1)
                  kubectl create secret docker-registry ecr-credentials \
                    --docker-server=${AWS_ACCOUNT}.dkr.ecr.us-east-1.amazonaws.com \
                    --docker-username=AWS \
                    --docker-password="${TOKEN}" \
                    --namespace=production \
                    --dry-run=client -o yaml | kubectl apply -f -
          restartPolicy: OnFailure
```

## Step 7: Verify Registry Access in Portainer

After configuration:

1. Go to a Kubernetes cluster in Portainer
2. Deploy an application using the **Form** method
3. In the **Image** field, the registry dropdown should only show permitted registries
4. Deploy and verify the pod pulls the image successfully:

```bash
kubectl get pods -n production
kubectl describe pod <pod-name> -n production | grep -A5 "Events:"
```

Successful pull shows: `Pulled: Successfully pulled image "registry.company.com/myapp:latest"`

## Step 8: Troubleshoot Pull Failures

```bash
# Check pod events for pull errors
kubectl describe pod <pod-name> -n production

# Common errors:
# ErrImagePull: Cannot pull image (auth or network issue)
# ImagePullBackOff: Retrying after ErrImagePull failure

# Test the pull secret
kubectl run test-pull \
  --image=registry.company.com/myapp:latest \
  --overrides='{"spec":{"imagePullSecrets":[{"name":"regcred"}]}}' \
  --restart=Never \
  -n production
```

## Conclusion

Configuring per-cluster registry access in Portainer for Kubernetes ensures each cluster can only use approved image sources. Combine Portainer's registry access controls with properly configured Kubernetes image pull secrets to create a complete and secure image distribution system. For AWS ECR, implement automated token refresh to prevent auth failures every 12 hours.
