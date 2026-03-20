# How to Set Up Image Policy in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, Image Policy, Policy Enforcement

Description: Implement image policies in Rancher to control which container images can be deployed, enforce security standards, and prevent use of untrusted images.

## Introduction

Image policies control which container images are allowed to run in your Kubernetes clusters. By enforcing image policies, you can prevent deployment of images from untrusted registries, ensure images are vulnerability-free, require image signing, and maintain compliance standards. This guide covers implementing image policies using admission webhooks, OPA, and Rancher's built-in tools.

## Prerequisites

- Rancher managing clusters with admission control support
- kubectl access with cluster-admin permissions
- Optional: Kubewarden, OPA Gatekeeper, or Kyverno installed

## Step 1: Enforce Registry Allow-listing with Kyverno

Install Kyverno for policy enforcement:

```bash
# Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace
```

Create a policy to restrict image registries:

```yaml
# allowed-registries-policy.yaml - Only allow images from trusted registries
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
  annotations:
    policies.kyverno.io/title: Restrict Image Registries
    policies.kyverno.io/description: >-
      Only allow container images from approved registries.
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: check-registry
      match:
        any:
          - resources:
              kinds:
                - Pod
      exclude:
        any:
          - resources:
              namespaces:
                - kube-system
                - kyverno
      validate:
        message: "Images must come from approved registries: registry.example.com, harbor.internal"
        pattern:
          spec:
            containers:
              # Allow only specific registry prefixes
              - image: "registry.example.com/* | harbor.internal/*"
            =(initContainers):
              - image: "registry.example.com/* | harbor.internal/*"
```

## Step 2: Block Latest Tag Usage

```yaml
# no-latest-tag-policy.yaml - Prevent use of mutable 'latest' tag
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations:
    policies.kyverno.io/title: Disallow Latest Tag
    policies.kyverno.io/description: >-
      Using :latest makes deployments unpredictable. Require explicit version tags.
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: require-image-tag
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Images must use a specific tag, not ':latest'"
        deny:
          conditions:
            any:
              # Deny if image has :latest tag or no tag at all
              - key: "{{request.object.spec.containers[].image}}"
                operator: AnyIn
                value:
                  - "*/latest"
                  - "*:latest"
```

## Step 3: Enforce Image Digest Pinning

```yaml
# require-digest-policy.yaml - Require images to be pinned by digest
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
  annotations:
    policies.kyverno.io/title: Require Image Digest
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-digest
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - production
      validate:
        message: "Production images must be pinned by digest (sha256:...)"
        foreach:
          - list: "request.object.spec.containers"
            deny:
              conditions:
                any:
                  # Deny if image doesn't contain @sha256:
                  - key: "{{ element.image }}"
                    operator: NotContains
                    value: "@sha256:"
```

## Step 4: Validate Vulnerability Scanning with Harbor

```yaml
# harbor-vulnerability-check.yaml - Policy to check Harbor scan results
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-vulnerability-scan
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-harbor-scan
      match:
        any:
          - resources:
              kinds:
                - Pod
      context:
        - name: imageScanResult
          apiCall:
            url: "https://harbor.internal/api/v2.0/projects/production/repositories/{{ imageNormalized(request.object.spec.containers[0].image) }}/artifacts?with_scan_overview=true"
            method: GET
            requestType: RawHTTP
            jmesPath: "[0].scan_overview.\"application/vnd.security.vulnerability.report; version=1.1\".summary.total"
      validate:
        message: "Image has unacceptable vulnerabilities"
        deny:
          conditions:
            any:
              - key: "{{ imageScanResult }}"
                operator: GreaterThan
                value: "0"
```

## Step 5: Enforce Image Signing with Cosign

Verify that images are signed using Sigstore/Cosign:

```yaml
# verify-image-signature.yaml - Enforce signed images
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-cosign-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - production
      verifyImages:
        - imageReferences:
            - "registry.example.com/production/*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/my-app/.github/workflows/release.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev
```

## Step 6: Using OPA Gatekeeper for Image Policies

```yaml
# gatekeeper-allowed-registries.yaml - OPA Gatekeeper constraint template
apiVersion: templates.gatekeeper.sh/v1beta1
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
          satisfied := [good | repo = input.parameters.repos[_] ; good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("Container image '%v' is not from an allowed repository.", [container.image])
        }
---
# Apply the constraint
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
  parameters:
    repos:
      - "registry.example.com/"
      - "harbor.internal/"
```

## Step 7: Audit Existing Workloads for Policy Compliance

```bash
# Find all images not from approved registries
kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | \
  grep -v "^registry.example.com\|^harbor.internal"

# Generate a Kyverno policy report
kubectl get policyreports --all-namespaces -o wide
```

## Step 8: Rancher OPA Integration

Enable OPA integration in Rancher:

1. Navigate to cluster **Apps** > **Charts**.
2. Install **OPA Gatekeeper** from the Rancher catalog.
3. Configure constraints through the Rancher UI.

## Conclusion

Image policies are a critical security control for production Kubernetes environments. Start with registry allow-listing to prevent image pulls from untrusted sources, then progressively add controls like tag restrictions, vulnerability scanning requirements, and image signing. Use Kyverno or OPA Gatekeeper through Rancher for a comprehensive policy enforcement approach, and regularly audit your clusters to catch policy violations before they become security incidents.
