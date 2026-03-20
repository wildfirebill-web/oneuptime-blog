# How to Enable OPA Gatekeeper Policies with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, OPA Gatekeeper, Kubernetes, Policy, Security

Description: Learn how to install OPA Gatekeeper and configure admission control policies for Kubernetes workloads managed through Portainer.

## What Is OPA Gatekeeper?

OPA (Open Policy Agent) Gatekeeper is a Kubernetes admission controller that enforces custom policies at the API level. Any request to the Kubernetes API (from Portainer or kubectl) must pass policy checks before being accepted.

## Installing OPA Gatekeeper

```bash
# Install Gatekeeper on your Kubernetes cluster

kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.14.0/deploy/gatekeeper.yaml

# Verify Gatekeeper is running
kubectl get pods -n gatekeeper-system
```

## Policy 1: Require Resource Limits on All Containers

```yaml
# constraint-template-required-resources.yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        # Block pods where containers don't have resource limits
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container <%v> must have CPU limits", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container <%v> must have memory limits", [container.name])
        }
---
# Apply the constraint to all pods
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

## Policy 2: Block Privileged Containers

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8spspprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspprivilegedcontainer

        violation[{"msg": msg}] {
          c := input.review.object.spec.containers[_]
          c.securityContext.privileged
          msg := sprintf("Privileged containers are not allowed: <%v>", [c.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: block-privileged
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

## Policy 3: Enforce Approved Image Registries

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-image-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    repos:
      - "registry.mycompany.com"    # Only allow internal registry
      - "gcr.io/myproject"          # Allowed GCP project
```

## Applying Policies via Portainer

Deploy Gatekeeper policies as Kubernetes manifests in Portainer:

1. Go to your Kubernetes environment.
2. Click the kubectl shell icon.
3. Apply the YAML manifests.

Or use the YAML manifest editor in **Applications**.

## Testing Policy Enforcement

```bash
# This deployment should be DENIED (no resource limits)
kubectl run test-pod --image=nginx --restart=Never

# Check Gatekeeper audit results
kubectl get constrainttemplate
kubectl get k8srequiredresources require-resource-limits \
  -o jsonpath='{.status.totalViolations}'
```

## Audit Mode vs. Enforce Mode

Start in audit mode to see violations without blocking deployments:

```yaml
spec:
  enforcementAction: warn   # "warn" = audit, "deny" = enforce
```

## Conclusion

OPA Gatekeeper provides programmable admission control for any Kubernetes workload managed through Portainer. Start with the most impactful policies (resource limits, no privileged containers, registry restrictions) and add more as your security posture matures.
