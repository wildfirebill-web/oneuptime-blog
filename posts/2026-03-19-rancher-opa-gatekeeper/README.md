# How to Set Up OPA Gatekeeper with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, OPA, Gatekeeper

Description: Learn how to deploy and configure OPA Gatekeeper in Rancher-managed clusters to enforce custom policies using the Rego policy language.

OPA Gatekeeper brings the power of the Open Policy Agent (OPA) to Kubernetes as a native admission controller. It lets you define custom policies using the Rego language and enforce them across your clusters. Rancher supports Gatekeeper deployment and management through its UI. This guide covers setting up Gatekeeper and creating policies.

## Prerequisites

- Rancher v2.5 or later
- kubectl access with cluster admin privileges
- Helm 3 installed
- Basic familiarity with Kubernetes admission controllers

## Step 1: Install OPA Gatekeeper

### Via Rancher UI

1. Navigate to the downstream cluster.
2. Go to **Apps & Marketplace** > **Charts**.
3. Search for **OPA Gatekeeper**.
4. Click **Install**.
5. Configure the namespace (default: `gatekeeper-system`).
6. Click **Install**.

### Via Helm

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update

helm install gatekeeper gatekeeper/gatekeeper \
  -n gatekeeper-system \
  --create-namespace \
  --set replicas=3 \
  --set audit.replicas=1
```

Verify the installation:

```bash
kubectl get pods -n gatekeeper-system
```

You should see the gatekeeper-controller-manager and gatekeeper-audit pods running.

## Step 2: Understand Gatekeeper Concepts

Gatekeeper uses two key resources:

- **ConstraintTemplate**: Defines a reusable policy using Rego. This creates a new CRD for the constraint.
- **Constraint**: An instance of a ConstraintTemplate applied to specific resources.

The workflow is: Create a ConstraintTemplate, then create Constraints based on it.

## Step 3: Create a Policy to Require Labels

Create a ConstraintTemplate that checks for required labels:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
```

Apply it:

```bash
kubectl apply -f required-labels-template.yaml
```

Create a Constraint that enforces the policy on namespaces:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-owner
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Namespace"]
    excludedNamespaces:
    - kube-system
    - gatekeeper-system
    - cattle-system
  parameters:
    labels:
    - "owner"
    - "environment"
```

Apply the constraint:

```bash
kubectl apply -f required-labels-constraint.yaml
```

## Step 4: Test the Policy

Try creating a namespace without the required labels:

```bash
kubectl create namespace test-no-labels
```

Expected output:

```plaintext
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
[ns-must-have-owner] Missing required labels: {"environment", "owner"}
```

Create a compliant namespace:

```bash
kubectl create namespace test-with-labels --dry-run=client -o yaml | \
  kubectl label -f - --dry-run=client -o yaml owner=team-a environment=staging | \
  kubectl apply -f -
```

## Step 5: Create a Policy to Block Privileged Containers

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sBlockPrivileged
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sblockprivileged

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged containers are not allowed: %v", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged init containers are not allowed: %v", [container.name])
        }
```

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockPrivileged
metadata:
  name: block-privileged-containers
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces:
    - kube-system
    - cattle-system
    - gatekeeper-system
```

## Step 6: Create a Policy to Restrict Image Registries

Only allow images from approved registries:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not startswith_any(container.image, input.parameters.repos)
          msg := sprintf("Container %v uses image %v which is not from an allowed repository", [container.name, container.image])
        }

        startswith_any(str, prefixes) {
          prefix := prefixes[_]
          startswith(str, prefix)
        }
```

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces:
    - kube-system
    - cattle-system
  parameters:
    repos:
    - "your-registry.example.com/"
    - "docker.io/library/"
```

## Step 7: Use Dry Run Mode for Testing

Before enforcing a policy, test it in dry run mode:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: test-required-labels
spec:
  enforcementAction: dryrun
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
  parameters:
    labels:
    - "app"
    - "version"
```

Check violations without blocking resources:

```bash
kubectl get k8srequiredlabels test-required-labels -o yaml
```

The `status.violations` section lists all resources that would be blocked.

## Step 8: Monitor Gatekeeper

View audit results:

```bash
kubectl get constraints -o yaml | grep -A 10 "violations"
```

Check Gatekeeper metrics:

```bash
kubectl get --raw /metrics -n gatekeeper-system | grep gatekeeper
```

Key metrics:
- `gatekeeper_violations`: Number of policy violations detected.
- `gatekeeper_constraint_template_status`: Status of constraint templates.

## Step 9: Use the Gatekeeper Policy Library

Instead of writing all policies from scratch, use the Gatekeeper policy library:

```bash
git clone https://github.com/open-policy-agent/gatekeeper-library.git
kubectl apply -f gatekeeper-library/library/
```

This provides pre-built templates for common policies like container limits, host networking restrictions, and read-only root filesystem requirements.

## Troubleshooting

### Constraint Template Not Syncing

Check the template status:

```bash
kubectl get constrainttemplate k8srequiredlabels -o yaml
```

Look for errors in the `status` section. Common issues include Rego syntax errors.

### Gatekeeper Blocking System Resources

Always exclude system namespaces in your constraints. If Gatekeeper is blocking critical resources, temporarily set enforcement to dryrun:

```bash
kubectl patch k8sblockprivileged block-privileged-containers \
  --type='merge' -p '{"spec":{"enforcementAction":"dryrun"}}'
```

## Conclusion

OPA Gatekeeper brings flexible, policy-as-code enforcement to Rancher-managed clusters. By defining ConstraintTemplates and Constraints, you can enforce a wide range of security and operational policies. Start with dry run mode, use the policy library for common patterns, and gradually move to enforcement as you gain confidence in your policies.
