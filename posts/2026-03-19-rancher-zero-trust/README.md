# How to Implement Zero Trust Security in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, Zero Trust

Description: Learn how to implement zero trust security principles in Rancher-managed Kubernetes clusters with network policies, mTLS, RBAC, and runtime verification.

Zero trust security operates on the principle that no entity, whether inside or outside the network, should be automatically trusted. Every access request must be verified. In a Kubernetes environment managed by Rancher, implementing zero trust requires layering multiple security controls. This guide covers the practical steps to achieve zero trust in your clusters.

## Prerequisites

- Rancher v2.5 or later
- Kubernetes clusters with a CNI that supports Network Policies
- kubectl and Helm 3 access
- Admin privileges on Rancher

## Step 1: Apply Default Deny Network Policies

The foundation of zero trust networking is denying all traffic by default. Apply default deny policies to every namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Automate this for all new namespaces by using a mutating webhook or OPA Gatekeeper constraint that requires default deny policies:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredefaultdeny
spec:
  crd:
    spec:
      names:
        kind: K8sRequireDefaultDeny
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredefaultdeny

        violation[{"msg": msg}] {
          input.review.kind.kind == "Namespace"
          input.review.operation == "CREATE"
          msg := "All namespaces must have a default deny NetworkPolicy. Apply one after creation."
        }
```

## Step 2: Enable Mutual TLS with a Service Mesh

Deploy a service mesh to encrypt all service-to-service communication with mTLS.

### Install Istio via Rancher

1. Navigate to the cluster in Rancher.
2. Go to **Apps & Marketplace** > **Charts**.
3. Search for **Istio**.
4. Click **Install** and configure options.
5. Enable **mTLS strict mode**.

### Configure Strict mTLS

After Istio is installed, enforce strict mTLS cluster-wide:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

Apply it:

```bash
kubectl apply -f strict-mtls.yaml
```

This ensures all communication between pods is encrypted and authenticated with mutual TLS.

## Step 3: Implement Least-Privilege RBAC

Create roles with minimal permissions for each team and service account:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-developer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "create", "update"]
```

Bind to specific users through Rancher:

1. Go to **Cluster Management** > **Members**.
2. Add users with specific roles.
3. Avoid granting cluster-admin to anyone who does not absolutely need it.

Audit existing RBAC to find overly permissive bindings:

```bash
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects[]'
```

## Step 4: Enforce Service Account Token Restrictions

Disable automatic mounting of service account tokens where not needed:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
automountServiceAccountToken: false
```

For pods that need API access, mount tokens explicitly with a limited audience:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false
  containers:
  - name: app
    image: my-app:latest
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: my-api
```

## Step 5: Implement Authorization Policies

With Istio, create fine-grained authorization policies:

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-server
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/production/sa/frontend"
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
```

This allows only the frontend service account to access the API endpoints, and only using GET and POST methods.

## Step 6: Enable Runtime Security Monitoring

Deploy Falco for runtime threat detection:

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  -n falco \
  --create-namespace \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true
```

Falco detects suspicious runtime behavior such as:
- Shell execution inside containers
- Unexpected network connections
- File access to sensitive paths
- Privilege escalation attempts

Create custom Falco rules for your environment:

```yaml
- rule: Unexpected Outbound Connection
  desc: Detect outbound connections to non-approved destinations
  condition: >
    outbound and not (fd.sip in (approved_ips))
  output: >
    Unexpected outbound connection (user=%user.name command=%proc.cmdline
    connection=%fd.name container=%container.name)
  priority: WARNING
```

## Step 7: Verify Image Integrity

Ensure only signed and verified images are deployed:

```bash
# Install Cosign

go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# Sign an image
cosign sign --key cosign.key your-registry/app:latest

# Verify an image
cosign verify --key cosign.pub your-registry/app:latest
```

Deploy a policy engine to enforce image verification:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8simageverification
spec:
  crd:
    spec:
      names:
        kind: K8sImageVerification
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8simageverification

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not startswith(container.image, "your-registry.example.com/")
          msg := sprintf("Image %v is not from the trusted registry", [container.image])
        }
```

## Step 8: Encrypt Data at Rest and in Transit

Ensure all data is encrypted:

- **In Transit**: mTLS via service mesh (Step 2).
- **At Rest**: Enable etcd encryption (see the encryption at rest guide).
- **Secrets**: Use external secrets management:

```bash
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace
```

## Step 9: Implement Continuous Verification

Set up continuous security scanning and policy evaluation:

```bash
# Install kube-bench for CIS benchmark checking
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
```

Schedule regular security assessments:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: security-audit
  namespace: security
spec:
  schedule: "0 6 * * 1"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: audit
            image: aquasec/kube-bench:latest
            command: ["kube-bench"]
          restartPolicy: OnFailure
```

## Zero Trust Checklist

- [ ] Default deny network policies on all namespaces
- [ ] Mutual TLS for all service communication
- [ ] Least-privilege RBAC for all users and service accounts
- [ ] Service account token restrictions
- [ ] Authorization policies for service-to-service access
- [ ] Runtime security monitoring (Falco)
- [ ] Image signing and verification
- [ ] Encryption at rest and in transit
- [ ] Continuous security scanning
- [ ] Audit logging for all API access

## Conclusion

Implementing zero trust in Rancher-managed Kubernetes clusters requires multiple complementary security layers. No single tool or configuration achieves zero trust alone. By combining default deny networking, mTLS, least-privilege RBAC, runtime monitoring, and continuous verification, you create an environment where every access request is verified and every anomaly is detected. Start with network policies and RBAC, then progressively add service mesh, runtime security, and image verification as your security maturity grows.
