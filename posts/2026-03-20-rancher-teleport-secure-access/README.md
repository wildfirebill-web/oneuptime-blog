# How to Configure Rancher with Teleport for Secure Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Teleport, Secure-access, Kubernetes, Zero-trust, PAM

Description: A guide to integrating Rancher with Teleport for zero-trust privileged access management, covering Kubernetes access, audit trails, and role-based access via Teleport.

## Overview

Teleport is an open-source privileged access management (PAM) platform that provides secure, audited access to Kubernetes clusters, SSH servers, databases, and web applications. Integrating Teleport with Rancher provides zero-trust access to Kubernetes clusters with full session recording, certificate-based authentication, and per-request authorization. This guide covers the integration setup.

## Architecture

```text
Developer/Operator
      |
   [MFA Authentication]
      |
   Teleport Proxy (public)
      |
   [Teleport Auth Server]
      |
   Teleport Kubernetes Agent (in each cluster)
      |
   Kubernetes API Server (via Rancher-managed cluster)
```

## Prerequisites

- Teleport v14+ (open source or enterprise)
- Rancher v2.7+ with managed Kubernetes clusters
- Valid TLS certificates for Teleport
- Identity provider (OIDC/SAML) configured in Teleport

## Step 1: Install Teleport Auth and Proxy

```yaml
# Teleport Helm values (teleport-cluster)

auth_service:
  enabled: true
  cluster_name: "teleport.company.com"

  # OIDC authentication (use your IdP)
  authentication:
    type: github   # or oidc, saml
    connector_name: github

proxy_service:
  enabled: true
  public_addr: "teleport.company.com:443"
  kube_public_addr: "teleport.company.com:3026"

kubernetes_service:
  enabled: true
```

```bash
# Install Teleport Cluster
helm repo add teleport https://charts.releases.teleport.dev
helm install teleport-cluster teleport/teleport-cluster \
  --namespace teleport \
  --create-namespace \
  --values teleport-values.yaml
```

## Step 2: Configure Teleport Kubernetes Access

### Connect Rancher-Managed Clusters to Teleport

Deploy the Teleport Kubernetes agent on each Rancher-managed cluster:

```yaml
# teleport-agent-values.yaml
roles: kube

authToken: "${TELEPORT_JOIN_TOKEN}"
proxyAddr: "teleport.company.com:443"

# Label this agent for role-based routing
labels:
  env: production
  cluster: prod-us-east-01
  managed-by: rancher
```

```bash
# Generate a join token for the agent
tctl tokens add --type=kube --ttl=1h

# Install Teleport agent on each Rancher cluster
helm install teleport-agent teleport/teleport-kube-agent \
  --namespace teleport \
  --create-namespace \
  --values teleport-agent-values.yaml
```

## Step 3: Create Teleport Kubernetes Roles

```yaml
# Teleport role for developers - limited access
kind: role
version: v6
metadata:
  name: k8s-developer
spec:
  allow:
    kubernetes_labels:
      env: ["staging", "development"]
    kubernetes_groups:
      - developers
    kubernetes_resources:
      - kind: pod
        name: "*"
        namespace: "*"
        verbs:
          - get
          - list
          - watch
      - kind: pod
        name: "*"
        namespace: "*"
        verbs:
          - exec    # Allow exec for debugging
  options:
    max_session_ttl: 8h
    require_session_mfa: "hardware_key"
---
# Teleport role for cluster admins
kind: role
version: v6
metadata:
  name: k8s-admin
spec:
  allow:
    kubernetes_labels:
      "*": "*"    # Access all clusters
    kubernetes_groups:
      - system:masters
    kubernetes_resources:
      - kind: "*"
        name: "*"
        namespace: "*"
        verbs: ["*"]
  options:
    max_session_ttl: 4h
    require_session_mfa: "hardware_key"
    record_session: true
```

## Step 4: Configure RBAC in Kubernetes

Map Teleport groups to Kubernetes RBAC:

```yaml
# ClusterRoleBinding: Map Teleport "developers" group to k8s role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: teleport-developers
subjects:
  - kind: Group
    name: developers    # Matches kubernetes_groups in Teleport role
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
---
# Allow pod exec for developers in specific namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: teleport-dev-exec
  namespace: development
subjects:
  - kind: Group
    name: developers
roleRef:
  kind: ClusterRole
  name: exec-pods    # Custom role allowing pod/exec
```

## Step 5: Configure Session Recording

```yaml
# Teleport auth server configuration - enable session recording
auth_service:
  session_recording: node   # or "proxy" for proxy-level recording
  enhanced_recording:
    enabled: true           # Record all commands within sessions
    command_buffer_size: 8096
    disk_buffer_size: 300
    cgroup_path: /cgroup2
```

## Step 6: Access Clusters via Teleport

```bash
# Developer workflow: Access production cluster
# Step 1: Login to Teleport with MFA
tsh login --proxy=teleport.company.com --user=jane.doe

# Step 2: List available clusters
tsh kube ls
# Output:
# Kube Cluster Name        Labels                  Selected
# ------------------------ ----------------------- --------
# prod-us-east-01          env=production           -
# staging-us-east-01       env=staging              -
# development-01           env=development          -

# Step 3: Switch to a cluster
tsh kube login prod-us-east-01

# Step 4: Use kubectl normally - all commands are audited
kubectl get pods -n my-app
kubectl logs -f pod/my-app-xxx

# Step 5: Session recording - all output is recorded
kubectl exec -it pod/my-app-xxx -- /bin/bash
```

## Step 7: Integration with Rancher SSO

You can configure Teleport to use the same identity provider as Rancher:

```yaml
# teleport.yaml - OIDC connector for same IdP as Rancher
connector_type: oidc
id: azure-ad
name: Azure AD (same as Rancher SSO)

redirect_url: https://teleport.company.com/v1/webapi/oidc/callback
client_id: "${AZURE_APP_CLIENT_ID}"
client_secret: "${AZURE_APP_CLIENT_SECRET}"
issuer_url: "https://login.microsoftonline.com/${TENANT_ID}/v2.0"

scope: ["openid", "email", "profile", "groups"]

claims_to_roles:
  - claim: groups
    value: "k8s-admins@company.com"
    roles: ["k8s-admin"]
  - claim: groups
    value: "k8s-developers@company.com"
    roles: ["k8s-developer"]
```

## Step 8: Audit and Compliance

Teleport records all Kubernetes API calls and shell sessions:

```bash
# View audit events for a user
tctl events search \
  --query='event=="kube.request" && user=="jane.doe"' \
  --from=2026-03-01 \
  --to=2026-03-19

# Export audit logs for compliance
tctl events export \
  --query='event=="session.upload"' \
  --format=json \
  > k8s-sessions-march.json
```

## Conclusion

Integrating Rancher with Teleport provides a zero-trust access layer over your Kubernetes clusters. Every access request requires authentication with MFA, all sessions are recorded for audit purposes, and access is scoped by roles that align with your identity provider's groups. This combination is particularly valuable for compliance-heavy environments (SOC 2, PCI DSS, HIPAA) that require detailed access logs and session recordings for every privileged operation.
