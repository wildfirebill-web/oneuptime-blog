# How to Restrict KubeShell to Admin Users in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Security, KubeShell, RBAC

Description: Learn how to restrict Portainer's KubeShell terminal access to administrator users only, preventing non-admin users from bypassing namespace-level access controls.

## Introduction

Portainer's KubeShell provides a browser-based kubectl terminal directly within the UI. While convenient, an unrestricted KubeShell can allow users to bypass namespace isolation and perform actions that exceed their intended permissions. Portainer Business Edition lets administrators restrict KubeShell access to admin users only.

## Prerequisites

- Portainer Business Edition (BE)
- Admin access to Portainer
- A Kubernetes environment configured in Portainer

## Why Restrict KubeShell?

The KubeShell runs with credentials scoped to the user's Portainer access level. However, the shell itself provides raw kubectl access, which could allow:

- Issuing commands across namespaces if RBAC is misconfigured
- Accessing cluster-level resources
- Running `kubectl exec` into pods the user shouldn't access
- Bypassing Portainer-enforced resource quotas through direct API calls

Restricting KubeShell to admins adds a clear security boundary.

## Step 1: Access Environment Security Settings

1. Log into Portainer as an **administrator**.
2. From the left sidebar, click **Environments**.
3. Click on your Kubernetes environment to open its configuration.

## Step 2: Restrict KubeShell to Admin Users

1. In the environment settings, scroll to the **Security** section.
2. Find the toggle labeled **Only allow administrators to use the KubeShell**.
3. Enable this toggle.
4. Click **Save environment settings**.

Once enabled:
- Admin users can still open KubeShell from **KubeShell** in the left sidebar
- Non-admin users will see the KubeShell option grayed out or hidden entirely
- Non-admin users can still use kubectl locally via downloaded kubeconfig files (if that access is granted)

## Step 3: Verify the Restriction Is Active

Log out and log back in as a **non-admin user** who has access to the Kubernetes environment:

1. Navigate to the Kubernetes environment.
2. Attempt to open the KubeShell.
3. The option should be hidden or produce an authorization error.

As an admin, you can test this without logging out by checking the UI behavior:

```bash
# Verify via Portainer API - check environment settings
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' | jq -r '.jwt')

# Get environment settings for endpoint ID 1
curl -s -H "Authorization: Bearer $TOKEN" \
  "https://portainer.example.com/api/endpoints/1" | \
  jq '.Kubernetes.Configuration.RestrictDefaultNamespace // empty'
```

## Step 4: Grant KubeShell Access Selectively

If you want specific non-admin users to have KubeShell access without full Portainer admin rights, consider this approach:

1. Create a **Portainer team** specifically for "cluster operators".
2. Assign these users to the team.
3. Grant the team **environment admin** access (not global admin) to the specific Kubernetes environment.
4. Environment admins can use KubeShell for that environment.

```
Portainer Role Hierarchy:
- Global Admin     → Full access to all environments + KubeShell everywhere
- Environment Admin → Full access to specific environment + KubeShell in that env
- Standard User    → Namespace-scoped access + NO KubeShell (when restricted)
```

## Step 5: Audit KubeShell Usage

Portainer BE logs KubeShell session access in the Activity Logs:

1. Go to **Settings** → **Authentication logs** (in BE).
2. Filter for KubeShell-related actions.
3. Review which admin users accessed the terminal and when.

You can also audit directly from the host:

```bash
# Check Portainer logs for KubeShell access
docker logs portainer 2>&1 | grep -i kubeshell | tail -20

# Or follow logs in real time
docker logs -f portainer 2>&1 | grep -i "kubeshell\|terminal\|exec"
```

## Alternative: Provide Scoped kubectl Access Instead

For users who need CLI access, provide scoped kubeconfig files instead of KubeShell:

1. Keep KubeShell restricted to admins.
2. Enable **kubectl access** in environment settings.
3. Allow users to download personal kubeconfig files scoped to their namespaces.
4. Users get familiar CLI access without the risk of unrestricted shell access.

```bash
# User downloads their scoped kubeconfig
# They can only access namespaces assigned to them in Portainer
kubectl get pods -n production    # Allowed
kubectl get nodes                 # Denied (cluster-level, not in their scope)
kubectl get pods -n other-team    # Denied (different namespace)
```

## Conclusion

Restricting KubeShell to admin users in Portainer is a straightforward configuration that significantly improves your Kubernetes security posture. Non-admin users can still get CLI access through scoped kubeconfig files, maintaining productivity while preventing accidental or intentional misuse of unrestricted shell access. Combine this with short kubeconfig token expiry and namespace isolation for a defense-in-depth approach.
