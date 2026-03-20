# How to Configure ArgoCD Application Sources with IPv6 Git URLs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, ArgoCD, GitOps, Git, Kubernetes, HTTPS

Description: Configure ArgoCD to connect to Git repositories over IPv6, including HTTPS and SSH repository URLs with IPv6 addresses, trust configuration, and troubleshooting connection issues.

## Introduction

ArgoCD connects to Git repositories as application sources for GitOps deployments. When Git servers are on IPv6 networks, ArgoCD must be configured to connect using IPv6 URLs. This involves adding repositories with IPv6 addresses in the URL, configuring TLS certificates for IPv6-addressed HTTPS endpoints, and ensuring the ArgoCD cluster's DNS can resolve Git hostnames to AAAA records.

## Add a Git Repository with IPv6 HTTPS URL

```bash
# ArgoCD CLI: add a Git repository over IPv6

argocd repo add "https://[2001:db8::git]:443/org/myrepo.git" \
    --username git \
    --password "$GIT_TOKEN" \
    --insecure-skip-server-verification    # Only for testing

# With TLS certificate verification (recommended for production)
argocd repo add "https://gitea.example.com/org/myrepo.git" \
    --tls-client-cert-path /tmp/git-ca.crt \
    --username git \
    --password "$GIT_TOKEN"
# Hostname must resolve to AAAA record for IPv6 connectivity

# List repositories
argocd repo list
```

## ArgoCD Repository Secret with IPv6

```yaml
# argocd-repo-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-repo-ipv6
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  # HTTPS repository with IPv6 address
  url: "https://[2001:db8::git]:443/org/myrepo.git"
  username: git
  password: "your-token"
  # TLS CA for the IPv6 Git server
  tlsClientCertData: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  insecure: "false"
```

```bash
kubectl apply -f argocd-repo-secret.yaml
```

## ArgoCD with SSH Git Repository over IPv6

```yaml
# argocd-ssh-repo.yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-ssh-ipv6
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  # SSH URL with IPv6 address (bracket notation in URL is NOT standard for SSH)
  # Use hostname that resolves to AAAA instead
  url: "ssh://git@gitserver.example.com/org/myrepo.git"
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
  insecure: "false"
```

```bash
# For SSH with IPv6, configure DNS to return AAAA record for gitserver.example.com
# OR configure the ArgoCD known_hosts for the IPv6 server

# Get SSH host key from IPv6 Git server
ssh-keyscan -6 2001:db8::git

# Add to ArgoCD known hosts
argocd cert add-ssh "[2001:db8::git]:22" --batch < /dev/stdin << 'EOF'
2001:db8::git ssh-rsa AAAAB3Nz...
EOF
```

## ArgoCD Application Using IPv6 Repository

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default

  source:
    # Repository added above with IPv6 URL
    repoURL: "https://[2001:db8::git]:443/org/myrepo.git"
    targetRevision: main
    path: kubernetes/overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## ArgoCD Network Configuration for IPv6

```yaml
# Ensure ArgoCD's repo-server pod can reach IPv6 Git servers
# Check the Kubernetes cluster's DNS supports AAAA records

# Test DNS resolution from ArgoCD pod
kubectl exec -n argocd deployment/argocd-repo-server -- \
    nslookup gitserver.example.com
# Should show IPv6 address

# Check connectivity to IPv6 Git server
kubectl exec -n argocd deployment/argocd-repo-server -- \
    curl -6 -v "https://[2001:db8::git]:443"

# If cluster is IPv6-only: ensure ArgoCD components use IPv6
# argocd-server ConfigMap:
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  # Force IPv6 for outbound connections
  server.listen: "[::]:8080"
```

## Configure ArgoCD cm for IPv6 Repo Server

```yaml
# argocd-cm ConfigMap additions
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # TLS configuration for IPv6 Git servers
  # Custom CA for self-signed certificates on IPv6 endpoints
  tls.certs.data: |
    -----BEGIN CERTIFICATE-----
    MIIBxxx...   # CA cert for IPv6 Git server
    -----END CERTIFICATE-----
```

## Troubleshoot ArgoCD IPv6 Git Connectivity

```bash
# Check ArgoCD repo-server logs for connection errors
kubectl logs -n argocd deployment/argocd-repo-server | grep -i "err\|ipv6\|connect"

# Test Git clone directly from repo-server pod
kubectl exec -n argocd deployment/argocd-repo-server -- \
    git ls-remote "https://[2001:db8::git]:443/org/myrepo.git"

# Check if IPv6 is available in the pod
kubectl exec -n argocd deployment/argocd-repo-server -- \
    ip -6 addr show

# Verify TLS certificate includes IPv6 SAN
openssl s_client -connect "[2001:db8::git]:443" -6 </dev/null 2>&1 | \
    openssl x509 -noout -text | grep -A3 "Subject Alternative"
# Should show: IP Address:2001:db8::git
```

## Conclusion

ArgoCD connects to Git repositories over IPv6 using HTTPS URLs with bracket notation (`https://[2001:db8::git]:443/`) or SSH URLs with hostnames resolving to AAAA records. Repository secrets reference these URLs and include TLS certificates or SSH keys. The ArgoCD `argocd-cm` ConfigMap accepts custom CA certificates for IPv6 Git servers with self-signed TLS. For SSH over IPv6, add the server's host key to ArgoCD's known_hosts using `argocd cert add-ssh`. Ensure the Kubernetes cluster's DNS returns AAAA records for Git hostnames when using hostnames rather than literal IPv6 addresses in repository URLs.
