# How to Configure ArgoCD Application Sources with IPv6 Git URLs - Repositories

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ArgoCD, GitOps, Kubernetes, Git, DevOps

Description: Configure ArgoCD to connect to Git repositories using IPv6 URLs, including SSH and HTTPS connections and repository credential management.

## Introduction

ArgoCD manages Kubernetes applications by syncing from Git repositories. When your Git server is only accessible over IPv6, or when you want to prefer IPv6 for repository connections, you need to configure ArgoCD's repository connections and ensure the underlying network supports IPv6.

## Prerequisites

- ArgoCD installed in an IPv6-capable Kubernetes cluster
- Git server accessible over IPv6 (GitLab, Gitea, or self-hosted)
- ArgoCD CLI installed

## Step 1: Verify ArgoCD Pods Have IPv6 Connectivity

```bash
# Check that ArgoCD pods are running in IPv6 cluster

kubectl get pods -n argocd -o wide

# Verify the argocd-server pod can reach IPv6 resources
kubectl exec -n argocd deployment/argocd-server -- \
    sh -c "ip -6 addr show && ping6 -c 2 2606:4700:4700::1111"

# Check ArgoCD's service addresses
kubectl get svc -n argocd
```

## Step 2: Add a Repository with IPv6 HTTPS URL

```bash
# Add a private Git repo accessible over IPv6 via HTTPS
argocd repo add https://[2001:db8::gitea]/org/repo.git \
    --username myuser \
    --password mypassword \
    --insecure-skip-server-verification  # Only for self-signed certs in testing

# Add with custom CA certificate (recommended for self-signed certs)
argocd repo add https://[2001:db8::gitea]/org/repo.git \
    --username myuser \
    --password mypassword \
    --tls-client-cert-path /tmp/git-ca.crt
```

## Step 3: Add a Repository with IPv6 SSH URL

```bash
# Add SSH key for GitHub/GitLab over IPv6
# First generate or provide an SSH key
ssh-keygen -t ed25519 -f /tmp/argocd-git-key -N ""

# Add the public key to your Git server

# Register the repo in ArgoCD using SSH with IPv6 address
argocd repo add git@[2001:db8::gitea]:org/repo.git \
    --ssh-private-key-path /tmp/argocd-git-key

# For GitHub (which has IPv6 connectivity):
argocd repo add git@github.com:org/repo.git \
    --ssh-private-key-path /tmp/argocd-git-key
```

## Step 4: Configure Repository via Kubernetes Secret

For GitOps-managed ArgoCD configuration:

```yaml
# argocd-repo-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-git-repo-ipv6
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: "https://[2001:db8::gitea]/org/repo.git"
  username: "git-user"
  password: "git-token"
  # Optional: TLS client certificate
  # tlsClientCertData: <base64-cert>
  # tlsClientCertKey: <base64-key>
```

```bash
kubectl apply -f argocd-repo-secret.yaml
```

## Step 5: Create an ArgoCD Application from IPv6 Source

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    # Use IPv6 URL for the Git repository
    repoURL: https://[2001:db8::gitea]/org/my-app.git
    targetRevision: HEAD
    path: k8s/

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f argocd-app.yaml

# Or via CLI
argocd app create my-app \
    --repo https://[2001:db8::gitea]/org/my-app.git \
    --path k8s/ \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace production \
    --sync-policy automated
```

## Step 6: Verify Repository Connectivity

```bash
# Check repository connection status
argocd repo list

# Get detailed status for a specific repo
argocd repo get https://[2001:db8::gitea]/org/repo.git

# Force a connection test
argocd repo refresh https://[2001:db8::gitea]/org/repo.git
```

## Troubleshooting IPv6 Repository Connections

```bash
# Check ArgoCD repo server logs for connection errors
kubectl logs -n argocd deployment/argocd-repo-server --tail=50 | grep -i "error\|ipv6\|connection"

# Test SSH connectivity to the Git server from the ArgoCD pod
kubectl exec -n argocd deployment/argocd-repo-server -- \
    ssh -T -o StrictHostKeyChecking=no git@2001:db8::gitea

# Test HTTPS connectivity
kubectl exec -n argocd deployment/argocd-repo-server -- \
    curl -6 -v https://[2001:db8::gitea]/api/v1/repos/org/repo
```

## Conclusion

ArgoCD supports IPv6 repository connections through standard Git URL notation with IPv6 addresses enclosed in square brackets. Repository credentials are stored as Kubernetes Secrets with the `argocd.argoproj.io/secret-type: repository` label. The key requirement is that ArgoCD pods in the cluster have IPv6 network connectivity, which is provided automatically in dual-stack or IPv6-only Kubernetes clusters. SSH and HTTPS repository connections both work identically with IPv6 URLs.
