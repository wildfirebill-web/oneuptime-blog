# How to Restrict Public Repository Usage in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Docker Hub, Registry Policy, Compliance

Description: Learn how to prevent Portainer users from pulling images from public registries and enforce use of approved private registries.

## Why Restrict Public Repositories?

Allowing unrestricted Docker Hub and public registry access introduces risks:

- **Unvetted images**: Public images may contain vulnerabilities or malware.
- **Supply chain attacks**: Typosquatted images (e.g., `nginxx` instead of `nginx`).
- **Compliance**: Some regulations require all images to be from approved sources.
- **Rate limiting**: Anonymous Docker Hub pulls are rate-limited.

## Restricting Public Images in Portainer

### Method 1: Disable Public Registries

1. Go to **Settings > Registries**.
2. Find **DockerHub** in the registry list.
3. Toggle it to **Off** or click **Disable**.

For the environment level:
1. Go to **Environments**, select your environment.
2. Under **Security**, disable **Allow users to pull images from public registries**.

### Method 2: Environment Security Settings

Some Portainer versions have a specific setting:

1. Go to **Environments > [Your Env] > Edit**.
2. Under **Security**, toggle **Allow users to use public images** to **Off**.

## Enforcing Registry Allow-Lists

If Portainer doesn't have a native allowlist, enforce at the Docker daemon level:

```json
// /etc/docker/daemon.json - restrict registries at the Docker level
{
  "registry-mirrors": [],
  "insecure-registries": [],
  // Block all registries except your approved ones
  "blocked-registries": ["docker.io", "ghcr.io", "gcr.io"],
  // Note: This may not be supported in all Docker versions
  // Alternative: Use OPA Gatekeeper or Cosign for policy enforcement
}
```

## Using OPA Gatekeeper to Enforce Registry Policy (Kubernetes)

```yaml
# OPA Gatekeeper ConstraintTemplate
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
          msg := sprintf("container <%v> has an invalid image repo <%v>", [container.name, container.image])
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
  parameters:
    repos:
      - "registry.mycompany.com"  # Only allow your private registry
      - "gcr.io/my-project"       # And specific GCP project
```

## Auditing Current Image Sources

```bash
# Find all unique image registries in use
docker ps --format '{{.Image}}' | \
  sed 's|/[^/]*:[^:]*$||' | \
  sort -u

# For Kubernetes
kubectl get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | \
  sort -u | grep -v "your-approved-registry"
```

## Conclusion

Restricting public repository usage is a key supply chain security control. Start with Portainer's registry settings to disable Docker Hub for non-admins, and layer in OPA Gatekeeper or image signing (Cosign) policies for Kubernetes environments requiring strict compliance.
