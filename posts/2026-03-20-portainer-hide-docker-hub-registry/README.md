# How to Hide Docker Hub from the Registry Dropdown in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker Hub, Registry, Configuration, DevOps

Description: Learn how to hide the Docker Hub option from Portainer's registry dropdown to enforce use of private registries.

## Introduction

By default, Portainer shows Docker Hub as an option in the registry dropdown when deploying containers or stacks. For organizations that require all images to come from private registries, or for air-gapped environments where Docker Hub is not accessible, hiding Docker Hub reduces confusion and prevents accidental pulls from public registries. This guide covers the configuration.

## Prerequisites

- Portainer CE or BE with admin access
- Understanding of why you want to restrict registry access

## Why Hide Docker Hub

Common reasons to hide Docker Hub:

1. **Security policy** — Organization requires all images to be scanned before use
2. **Air-gapped environments** — No internet access; Docker Hub is unreachable
3. **Compliance** — All deployed software must be from approved registries
4. **Standardization** — Enforce use of internal registry mirror
5. **Rate limiting** — Prevent unauthenticated pulls that hit Docker Hub limits

## Step 1: Access Registry Settings

1. Log in to Portainer as admin
2. Click **Settings** in the sidebar
3. Scroll to the **Registry settings** section

Or navigate to:

1. **Registries** → Find Docker Hub entry → Edit

## Step 2: Hide Docker Hub

In Portainer BE:

1. Go to **Registries**
2. Find the **Docker Hub** entry
3. Click **Edit**
4. Disable or restrict Docker Hub visibility

In some Portainer versions, you can completely remove Docker Hub from the list by not adding credentials and configuring it to be hidden.

## Alternative: Configure via Portainer API

```bash
# Get the admin JWT token
TOKEN=$(curl -s -X POST http://portainer:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r .jwt)

# List registries to find Docker Hub ID
curl -H "Authorization: Bearer $TOKEN" \
  http://portainer:9000/api/registries

# Update Docker Hub registry to be restricted
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"Name":"Docker Hub","URL":"https://index.docker.io","TeamAccessPolicies":{}}' \
  http://portainer:9000/api/registries/1
```

## Step 3: Restrict Docker Hub via Team Access (BE)

In Portainer Business Edition, use team-based registry access to restrict Docker Hub:

1. Go to **Registries → Docker Hub**
2. Click **Manage access**
3. Remove all team access or set to "No access"
4. Only admins will see Docker Hub in the dropdown

## Step 4: Configure Docker Daemon to Block Docker Hub

For a more robust block (regardless of Portainer UI), configure the Docker daemon:

```json
// /etc/docker/daemon.json - Block Docker Hub entirely
{
  "registry-mirrors": ["https://your-internal-mirror.company.com"],
  "insecure-registries": [],
  "live-restore": true
}
```

Block Docker Hub at the network level using a firewall rule:

```bash
# Block Docker Hub IP ranges (use with caution - IPs change)
# Better to use a local proxy/mirror that blocks
```

## Step 5: Use a Registry Mirror as Replacement

Instead of just hiding Docker Hub, replace it with an internal mirror:

```json
// /etc/docker/daemon.json
{
  "registry-mirrors": ["https://internal-mirror.company.com"]
}
```

In Portainer:

1. Add your internal mirror as a Custom registry
2. Hide Docker Hub (it won't be needed since the mirror serves Docker Hub images)

## Step 6: Policy Documentation

When hiding Docker Hub, document the policy for developers:

```markdown
## Container Registry Policy

All container images must be pulled from the internal registry:
- Internal registry: registry.company.com
- All external images are mirrored here after security scanning
- To add a new external image: submit request to ops@company.com
- Docker Hub is disabled in Portainer to enforce this policy
```

## Step 7: Test the Configuration

After hiding Docker Hub:

1. Log in as a non-admin user
2. Go to **Containers → Add container**
3. Click the registry dropdown
4. Docker Hub should not appear (only your private registry should show)
5. Test that pulling an image from your private registry works

## Considering the Trade-offs

| Aspect | Hiding Docker Hub | Blocking at Network Level |
|--------|------------------|--------------------------|
| Ease of implementation | Simple | Requires firewall config |
| Effectiveness | UI-only (CLI still works) | Blocks all pulls |
| User experience | Cleaner UI | Fails with error |
| Maintenance | Low | Medium |

For strict enforcement, combine both: hide in Portainer UI AND block at the network/firewall level.

## Conclusion

Hiding Docker Hub from Portainer's registry dropdown is a straightforward way to guide users toward approved internal registries. Combined with a properly configured internal registry mirror and network-level controls, you can ensure all container images in your organization go through your security approval process before deployment. This is an important control for regulated industries and security-conscious organizations.
