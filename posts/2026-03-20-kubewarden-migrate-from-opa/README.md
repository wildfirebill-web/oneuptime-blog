# How to Migrate from OPA Gatekeeper to Kubewarden

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubewarden, OPA, Gatekeeper, Migration, Policy as Code, Kubernetes, Admission Control, SUSE Rancher

Description: Learn how to migrate Kubernetes admission policies from OPA Gatekeeper to Kubewarden, including mapping ConstraintTemplates to ClusterAdmissionPolicies and translating Rego logic to supported...

---

Migrating from OPA Gatekeeper to Kubewarden involves three main steps: understanding the policy mapping, rewriting policy logic in a supported language (Rust, Go, or others), and deploying the new policies while removing old ones safely.

---

## Architecture Comparison

| Concept | OPA Gatekeeper | Kubewarden |
|---|---|---|
| Policy language | Rego | Rust, Go, AssemblyScript, Swift (WASM) |
| Policy packaging | ConstraintTemplate CRD | OCI-packaged WASM module |
| Policy instance | Constraint CRD | ClusterAdmissionPolicy / AdmissionPolicy |
| Policy distribution | In-cluster | OCI registry |
| Testing tool | conftest | kwctl |

---

## Step 1: Inventory Existing Gatekeeper Policies

```bash
# List all ConstraintTemplates

kubectl get constrainttemplate

# List all Constraint instances
kubectl get constraints -A

# Export all constraints for review
kubectl get constrainttemplate -o yaml > gatekeeper-templates.yaml
kubectl get constraints -A -o yaml > gatekeeper-constraints.yaml
```

---

## Step 2: Map Rego Logic to a Kubewarden Policy

Take a simple Gatekeeper ConstraintTemplate that requires resource limits and translate it:

**OPA Gatekeeper (Rego):**

```rego
package k8srequiredlimits

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  not container.resources.limits.cpu
  msg := sprintf("Container '%v' is missing CPU limit", [container.name])
}
```

**Kubewarden equivalent (Go):**

```go
// main.go
func validate(payload []byte) ([]byte, error) {
    var request kubewarden_protocol.ValidationRequest
    json.Unmarshal(payload, &request)

    pod := &corev1.Pod{}
    json.Unmarshal(request.Request.Object.Raw, pod)

    for _, container := range pod.Spec.Containers {
        if container.Resources.Limits == nil ||
           container.Resources.Limits.Cpu().IsZero() {
            return kubewarden.RejectRequest(
                kubewarden.Message(fmt.Sprintf(
                    "Container '%s' is missing CPU limit", container.Name)),
                kubewarden.Code(400),
            )
        }
    }
    return kubewarden.AcceptRequest()
}
```

---

## Step 3: Check the Kubewarden Policy Hub First

Before writing custom policies, check if an equivalent policy already exists:

```bash
# Search the Kubewarden Policy Hub
kwctl search cpu-limits

# Pull and inspect an existing policy
kwctl pull registry://ghcr.io/kubewarden/policies/require-pod-resources:latest
kwctl inspect registry://ghcr.io/kubewarden/policies/require-pod-resources:latest
```

Many common Gatekeeper policies (PSP replacements, label enforcement, image restrictions) already have Kubewarden equivalents in the Hub.

---

## Step 4: Deploy Kubewarden Policies in Monitor Mode

Before removing Gatekeeper, run Kubewarden policies in monitor mode to verify they behave correctly:

```yaml
# disallow-privileged-kubewarden.yaml
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: disallow-privileged
spec:
  module: ghcr.io/kubewarden/policies/pod-privileged:v0.2.5
  mode: monitor    # Logs violations without blocking
  rules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      resources: ["pods"]
      operations: ["CREATE", "UPDATE"]
  mutating: false
```

```bash
kubectl apply -f disallow-privileged-kubewarden.yaml

# Check policy logs in monitor mode
kubectl logs -n kubewarden deployment/kubewarden-policy-server-default \
  | grep "disallow-privileged"
```

---

## Step 5: Validate Coverage

```bash
# Check Kubewarden audit scan results
kubectl get policyreport -A

# Confirm all violations that Gatekeeper was catching
# are now also caught by Kubewarden in monitor mode
kubectl get policyreport -A -o json | \
  jq '.items[].results[] | select(.result == "fail") | .policy'
```

---

## Step 6: Switch Kubewarden to Enforce Mode

Once monitor mode confirms correct behavior, switch to enforce:

```bash
kubectl patch clusteradmissionpolicy disallow-privileged \
  --type merge \
  -p '{"spec":{"mode":"protect"}}'
```

---

## Step 7: Remove Gatekeeper

```bash
# Delete all Constraints first (remove enforcement)
kubectl delete constraints --all -A

# Delete all ConstraintTemplates
kubectl delete constrainttemplate --all

# Uninstall Gatekeeper
helm uninstall gatekeeper -n gatekeeper-system

# Remove the namespace
kubectl delete namespace gatekeeper-system
```

---

## Migration Timeline

```text
Week 1: Inventory Gatekeeper policies → identify Hub equivalents
Week 2: Write custom policies for gaps → test with kwctl
Week 3: Deploy Kubewarden in monitor mode alongside Gatekeeper
Week 4: Validate coverage → switch to enforce → remove Gatekeeper
```

---

## Best Practices

- Always run Kubewarden in `monitor` mode before removing Gatekeeper - this ensures no coverage gap.
- Use the Kubewarden Policy Hub to find ready-made replacements for common Gatekeeper policies.
- Rewriting Rego in Go is usually the most straightforward migration path for teams familiar with Go.
