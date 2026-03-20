# How to Set Up ArgoCD with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ArgoCD, Rancher, GitOps, Kubernetes, Continuous Delivery, Helm, Deployment

Description: Learn how to install ArgoCD on a Rancher-managed cluster and configure it to deploy applications via GitOps, including multi-cluster app delivery and RBAC integration.

---

ArgoCD brings declarative GitOps to Kubernetes. When deployed on a Rancher-managed cluster, you get the best of both worlds: ArgoCD's powerful sync engine and Rancher's cluster management UI.

---

## Step 1: Install ArgoCD via Helm

Use Helm to install ArgoCD into its own namespace:

```bash
# Add the ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD with high-availability replicas
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --set server.replicas=2 \
  --set repoServer.replicas=2 \
  --set applicationSet.replicaCount=2
```

---

## Step 2: Expose the ArgoCD API Server

Create an ingress to access the ArgoCD UI via Rancher's ingress controller:

```yaml
# argocd-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    # Disable SSL redirect since ArgoCD handles TLS itself
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
```

```bash
kubectl apply -f argocd-ingress.yaml
```

---

## Step 3: Log In and Change Default Password

```bash
# Get the initial admin password (stored in a secret)
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath='{.data.password}' | base64 -d

# Log in with the CLI
argocd login argocd.example.com \
  --username admin \
  --password <initial-password>

# Change the password
argocd account update-password
```

---

## Step 4: Register a Downstream Rancher Cluster

ArgoCD can deploy to clusters other than the one it runs on. Register a Rancher-managed cluster:

```bash
# List available contexts in your kubeconfig
kubectl config get-contexts

# Add the downstream cluster to ArgoCD
argocd cluster add rancher-downstream \
  --name production-cluster
```

---

## Step 5: Create an Application

This Application manifest tells ArgoCD to sync a Helm chart from a Git repository to the production cluster:

```yaml
# my-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/my-app.git
    targetRevision: main
    path: helm/my-app
    helm:
      valueFiles:
        - values-production.yaml
  destination:
    server: https://production-cluster.rancher.example.com
    namespace: my-app
  syncPolicy:
    automated:
      # Automatically delete resources removed from Git
      prune: true
      # Automatically re-sync if the cluster drifts from Git
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f my-app.yaml
```

---

## Step 6: Configure Rancher SSO for ArgoCD

Connect ArgoCD to Rancher's authentication by adding an OIDC connector in ArgoCD's `argocd-cm` ConfigMap:

```yaml
# Patch argocd-cm to add OIDC with Rancher's Dex
data:
  url: https://argocd.example.com
  oidc.config: |
    name: Rancher
    issuer: https://rancher.example.com/v1/oidc
    clientID: argocd
    clientSecret: $oidc.rancher.clientSecret
    requestedScopes:
      - openid
      - profile
      - email
      - groups
```

---

## Best Practices

- Use ArgoCD **Projects** to scope which repos and clusters each team can access.
- Enable **resource health checks** in ArgoCD for Rancher CRDs like `ClusterV3`.
- Pair ArgoCD with **Rancher Fleet** for different layers: Fleet for cluster-level config, ArgoCD for app delivery.
