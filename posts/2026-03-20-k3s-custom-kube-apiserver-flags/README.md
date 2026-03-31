# How to Configure K3s with Custom kube-apiserver Flags - Kube

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: k3s, Kube-apiserver, Configuration, Kubernetes, Security, SUSE Rancher

Description: Learn how to pass custom kube-apiserver flags to K3s for advanced configuration including feature gates, admission plugins, audit logging, and API server tuning.

---

K3s embeds the kube-apiserver and other Kubernetes components. You can pass custom flags to the embedded kube-apiserver through K3s configuration to enable feature gates, customize admission controllers, and tune API server behavior.

---

## Step 1: Pass kube-apiserver Flags via Config File

```yaml
# /etc/rancher/k3s/config.yaml

kube-apiserver-arg:
  - "enable-admission-plugins=NodeRestriction,PodSecurity"
  - "audit-log-path=/var/log/kubernetes/audit.log"
  - "audit-log-maxage=30"
  - "audit-log-maxbackup=10"
  - "audit-log-maxsize=100"
  - "feature-gates=EphemeralContainers=true"
  - "anonymous-auth=false"
  - "profiling=false"
```

Restart K3s to apply:

```bash
systemctl restart k3s
```

---

## Step 2: Enable Pod Security Admission

```yaml
# /etc/rancher/k3s/config.yaml
kube-apiserver-arg:
  - "enable-admission-plugins=NodeRestriction,PodSecurity"

# Create the Pod Security configuration file
# /etc/rancher/k3s/pod-security-admission.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: PodSecurity
    configuration:
      apiVersion: pod-security.admission.config.k8s.io/v1
      kind: PodSecurityConfiguration
      defaults:
        enforce: "baseline"
        enforce-version: "latest"
        audit: "restricted"
        warn: "restricted"
      exemptions:
        namespaces:
          - kube-system
          - cattle-system
```

```yaml
# /etc/rancher/k3s/config.yaml - reference the config file
kube-apiserver-arg:
  - "admission-control-config-file=/etc/rancher/k3s/pod-security-admission.yaml"
  - "enable-admission-plugins=NodeRestriction,PodSecurity"
```

---

## Step 3: Configure Audit Logging

```yaml
# /etc/rancher/k3s/config.yaml
kube-apiserver-arg:
  - "audit-log-path=/var/log/k3s/audit.log"
  - "audit-log-maxage=30"
  - "audit-log-maxbackup=5"
  - "audit-log-maxsize=100"
  - "audit-policy-file=/etc/rancher/k3s/audit-policy.yaml"
```

```yaml
# /etc/rancher/k3s/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["pods"]
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
```

---

## Step 4: Enable Feature Gates

```yaml
# /etc/rancher/k3s/config.yaml
kube-apiserver-arg:
  - "feature-gates=EphemeralContainers=true,ServerSideApply=true"

kube-controller-manager-arg:
  - "feature-gates=EphemeralContainers=true"

kubelet-arg:
  - "feature-gates=EphemeralContainers=true"
```

---

## Step 5: Tune API Server Performance

```yaml
# /etc/rancher/k3s/config.yaml
kube-apiserver-arg:
  # Increase max requests in flight for high-load clusters
  - "max-requests-inflight=800"
  - "max-mutating-requests-inflight=400"

  # Increase watch cache size
  - "watch-cache-sizes=pods#1000,nodes#100"

  # Set API server priority and fairness
  - "enable-priority-and-fairness=true"
```

---

## Step 6: Configure OIDC Authentication

```yaml
# /etc/rancher/k3s/config.yaml
kube-apiserver-arg:
  - "oidc-issuer-url=https://accounts.google.com"
  - "oidc-client-id=my-k8s-client"
  - "oidc-username-claim=email"
  - "oidc-groups-claim=groups"
```

---

## Step 7: Verify Applied Flags

```bash
# Check the running kube-apiserver process arguments
ps aux | grep kube-apiserver | tr ' ' '\n' | grep -E "^\-\-"

# Or check the API server pod (K3s embeds it, but you can see it via)
kubectl get pod -n kube-system -l component=kube-apiserver -o yaml 2>/dev/null

# For K3s, inspect the process directly
cat /proc/$(pidof k3s server)/cmdline | tr '\0' '\n' | grep apiserver
```

---

## Available Flag Categories

| Category | Example Flags |
|---|---|
| Security | `--anonymous-auth`, `--enable-admission-plugins` |
| Audit | `--audit-log-path`, `--audit-policy-file` |
| Authentication | `--oidc-issuer-url`, `--token-auth-file` |
| Performance | `--max-requests-inflight`, `--watch-cache-sizes` |
| Feature gates | `--feature-gates` |

---

## Best Practices

- Pass all kube-apiserver customizations through `/etc/rancher/k3s/config.yaml` under `kube-apiserver-arg` - this is the supported way and survives K3s upgrades.
- Enable audit logging (`--audit-log-path`) on any cluster used for production or compliance - it creates a record of all API server operations.
- Test new kube-apiserver flags in a K3d local cluster before applying them to production - some flags can break API server startup if misconfigured.
