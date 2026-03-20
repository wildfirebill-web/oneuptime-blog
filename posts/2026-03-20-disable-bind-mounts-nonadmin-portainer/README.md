# How to Disable Bind Mounts for Non-Admin Users in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Bind Mounts, Container Security, Hardening

Description: Learn how to prevent non-administrator Portainer users from creating containers with host directory bind mounts.

## Why Disable Bind Mounts?

Bind mounts let a container access host filesystem directories directly. In the wrong hands, this is a serious security risk:

- A developer could mount `/etc/passwd` and read system credentials.
- A malicious container could mount `/var/run/docker.sock` and escape the container.
- A mount on `/` gives the container full read access to the host.

Portainer allows administrators to disable bind mount capability for non-admin users.

## Disabling Bind Mounts in Portainer

### For Docker Standalone/Swarm Environments

1. In Portainer, go to **Environments**.
2. Click on your Docker environment.
3. Click **Edit**.
4. Under **Security** or **Policies**, find **Allow bind mounts**.
5. Toggle it **Off**.
6. Click **Update environment**.

### For Kubernetes Environments

1. Go to the Kubernetes environment settings.
2. Under **Cluster > Setup**, find the **Security** section.
3. Disable **Allow users to use bind mounts** or similar options.

## What Happens When Bind Mounts Are Disabled

Non-admin users who attempt to create a container with a bind mount will see an error:

```text
Error: Bind mounts are not allowed for this environment.
```

Admins can still create bind mounts - the restriction applies only to standard users.

## Allowing Specific Named Volumes Instead

Redirect users from bind mounts to named volumes, which are safer:

```yaml
# Container with a named volume (allowed)

services:
  app:
    image: myapp:latest
    volumes:
      - app-data:/data          # Named volume - SAFE

# Instead of a bind mount (blocked)
      # - /host/path:/data      # Bind mount - BLOCKED
```

## Verifying the Restriction via API

```bash
# Attempt to create a container with a bind mount as a non-admin user
# Should return an authorization error
curl -X POST "${PORTAINER_URL}/api/endpoints/1/docker/containers/create" \
  -H "Authorization: Bearer ${NON_ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "Image": "nginx:alpine",
    "HostConfig": {
      "Binds": ["/host/path:/container/path"]
    }
  }'
# Expected: 403 Forbidden or authorization error
```

## When to Allow Bind Mounts

Bind mounts are legitimate in some scenarios:

- Development environments where developers need host filesystem access.
- Log collection agents that must read `/var/log`.
- Monitoring agents that need `/proc` or `/sys` access.

In these cases, consider:
1. Restricting bind mounts to specific, approved paths.
2. Using trusted (admin) users for these deployments.
3. Deploying log agents via DaemonSets with appropriate privileges.

## Conclusion

Disabling bind mounts for non-admin users in Portainer prevents a class of container escape and privilege escalation attacks. Combined with restricting host networking and privileged mode, this forms a strong baseline security posture for multi-user Portainer installations.
