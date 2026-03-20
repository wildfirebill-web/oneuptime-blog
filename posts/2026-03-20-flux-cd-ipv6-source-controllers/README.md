# How to Configure Flux CD Source Controllers with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Flux CD, GitOps, Kubernetes, Source Controller, HelmRepository

Description: Configure Flux CD source controllers (GitRepository, HelmRepository, OCIRepository) to pull from IPv6-addressed Git servers, Helm registries, and OCI repositories in dual-stack or IPv6-only Kubernetes clusters.

## Introduction

Flux CD uses Source Controllers to synchronize configuration from Git repositories, Helm chart repositories, and OCI registries. When these sources are hosted on IPv6 networks, Flux Source Controllers must connect via IPv6. Configuration involves creating source objects with IPv6 URLs, configuring TLS trust for IPv6 endpoints, and ensuring Flux components bind to IPv6 in the cluster network.

## GitRepository with IPv6 HTTPS Endpoint

```yaml
# git-repo-ipv6.yaml

# Secret with Git credentials for IPv6 server
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
  namespace: flux-system
type: Opaque
stringData:
  username: git
  password: "your-access-token"

---
# GitRepository source pointing to IPv6 Git server
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  # URL with IPv6 literal address
  url: "https://[2001:db8::git]:443/org/myapp.git"
  ref:
    branch: main
  secretRef:
    name: git-credentials
  verify:
    provider: cosign  # Optional: verify commits
```

```bash
# Apply the source
kubectl apply -f git-repo-ipv6.yaml

# Watch the source controller reconcile
flux get source git myapp

# Check for errors
kubectl describe gitrepository myapp -n flux-system | grep -A10 "Status:"
```

## GitRepository with SSH over IPv6

```yaml
# flux-ssh-source.yaml

apiVersion: v1
kind: Secret
metadata:
  name: git-ssh-key
  namespace: flux-system
type: Opaque
stringData:
  identity: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
  identity.pub: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5..."
  known_hosts: |
    [2001:db8::git]:22 ssh-rsa AAAAB3NzaC1...

---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp-ssh
  namespace: flux-system
spec:
  interval: 5m
  url: "ssh://git@[2001:db8::git]/org/myapp.git"
  ref:
    branch: main
  secretRef:
    name: git-ssh-key
```

```bash
# Get the SSH host key from the IPv6 Git server
# (run from a machine that can reach the server)
ssh-keyscan -6 -p 22 2001:db8::git

# Add to the known_hosts in the secret above
```

## HelmRepository with IPv6 Endpoint

```yaml
# helm-repo-ipv6.yaml

apiVersion: v1
kind: Secret
metadata:
  name: helm-credentials
  namespace: flux-system
type: Opaque
stringData:
  username: helm-user
  password: "helm-password"
  # CA certificate for the IPv6 Helm repo
  caFile: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: internal-charts
  namespace: flux-system
spec:
  interval: 10m
  url: "https://[2001:db8::helm]:443/charts"
  secretRef:
    name: helm-credentials
```

## OCIRepository with IPv6 Registry

```yaml
# oci-source-ipv6.yaml

apiVersion: v1
kind: Secret
metadata:
  name: oci-credentials
  namespace: flux-system
type: Opaque
stringData:
  username: registry-user
  password: "registry-password"

---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: myapp-charts
  namespace: flux-system
spec:
  interval: 5m
  url: "oci://[2001:db8::registry]:5000/myapp"
  ref:
    semver: ">=1.0.0"
  secretRef:
    name: oci-credentials
  verify:
    provider: cosign
```

## Flux Kustomization Using IPv6 Source

```yaml
# kustomization-ipv6.yaml

apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: myapp             # References the GitRepository above
  path: "./kubernetes/production"
  prune: true
  timeout: 5m
  # Health checks for the deployed resources
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp
      namespace: production
```

## Verify Flux Source Controllers on IPv6

```bash
# Check all Flux sources
flux get sources all -n flux-system

# Watch GitRepository sync status
flux get source git -A --watch

# Check source controller logs for IPv6 connection attempts
kubectl logs -n flux-system deployment/source-controller | \
    grep -E "ipv6|2001:|fd00:|connect|error" | tail -30

# Test connectivity from source-controller pod
kubectl exec -n flux-system deployment/source-controller -- \
    curl -6 -I "https://[2001:db8::git]:443"

# Check if source-controller has IPv6
kubectl exec -n flux-system deployment/source-controller -- \
    ip -6 addr show
```

## Flux in IPv6-Only Kubernetes Cluster

```yaml
# Ensure Flux components listen on IPv6 in IPv6-only clusters
# Install Flux with IPv6 support
flux install \
    --components-extra=image-reflector-controller,image-automation-controller

# Check Flux service endpoints
kubectl get svc -n flux-system
# source-controller should have IPv6 ClusterIP if cluster is dual-stack

# flux-system namespace services should bind to IPv6
kubectl get endpoints -n flux-system source-controller -o yaml | grep -i ipv6
```

## Troubleshoot Flux IPv6 Source Issues

```bash
# Source not ready: check status
kubectl describe gitrepository myapp -n flux-system

# Common errors:
# "x509: certificate is not valid for any of..."
# → TLS cert missing IPv6 SAN IP
# Fix: regenerate cert with IP SAN for 2001:db8::git

# "dial tcp [2001:db8::git]:443: connect: connection refused"
# → Server not listening on IPv6
# Fix: check Git server binds to [::]:443

# "no such host" for hostname-based URLs
# → DNS not returning AAAA record
# Fix: add AAAA record to DNS, or use literal IPv6 in URL

# Reconcile manually to force retry
flux reconcile source git myapp -n flux-system
```

## Conclusion

Flux CD Source Controllers connect to IPv6 Git servers, Helm registries, and OCI registries using URLs with bracketed IPv6 literals (`https://[2001:db8::git]:443/`) or hostnames with AAAA DNS records. The `known_hosts` field in SSH secrets must include the IPv6 address of the Git server (using `[ipv6]:port` format from `ssh-keyscan`). TLS certificates for IPv6 HTTPS endpoints must include the IPv6 address as a Subject Alternative Name (SAN) IP entry. The source controller pod must have IPv6 access in the cluster network — verify with `kubectl exec` and `ip -6 addr show` inside the pod. For IPv6-only clusters, ensure Flux's services and endpoints receive IPv6 addresses from the cluster's dual-stack or IPv6-only CNI.
