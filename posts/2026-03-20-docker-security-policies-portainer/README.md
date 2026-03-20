# How to Set Up Docker Security Policies in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Docker, Policies, Container Security

Description: Learn how to configure comprehensive Docker security policies in Portainer to enforce a secure baseline for all container deployments.

## Overview of Docker Security Policies in Portainer

Portainer's environment security settings let administrators define a security baseline that non-admin users cannot override. These policies are enforced at the API level — even if a user crafts a direct Docker API call, Portainer rejects it.

## Accessing Security Settings

1. Go to **Environments** in Portainer.
2. Select your Docker environment.
3. Click **Edit**.
4. Scroll to the **Security** section.

## Recommended Security Policy Configuration

### Disable Privileged Containers

Privileged containers have full access to the host kernel. Disable for all non-admin users:

```yaml
# What a privileged container can do (and why it's dangerous)
docker run --privileged ubuntu \
  mount /dev/sda1 /mnt  # Mount host disk
```

Toggle: **Allow containers to run in privileged mode** → **OFF**

### Disable Host Namespace Access

Prevent access to host-level namespaces:

Toggle **OFF**:
- Allow container to use the host PID namespace
- Allow container to use the host IPC namespace
- Allow container to use the host network mode

### Disable Bind Mounts

Prevent host filesystem exposure via bind mounts.

Toggle: **Allow bind mounts** → **OFF**

### Restrict Docker Socket Access

Never allow mounting `/var/run/docker.sock` in containers — it provides Docker daemon control:

```bash
# This is effectively root on the host - should never be allowed
docker run -v /var/run/docker.sock:/var/run/docker.sock ubuntu
```

## Applying Linux Capabilities Restrictions

Use Seccomp profiles and capability restrictions for additional hardening:

```yaml
# docker-compose.yml with restricted capabilities
services:
  app:
    image: my-app:latest
    security_opt:
      - no-new-privileges:true     # Prevent privilege escalation
      - seccomp:default            # Apply default seccomp profile
    cap_drop:
      - ALL                        # Drop ALL capabilities
    cap_add:
      - NET_BIND_SERVICE           # Only add what's needed
    read_only: true                # Read-only root filesystem
    tmpfs:
      - /tmp                       # Writable tmp in memory
```

## Per-User Policy Exceptions

For trusted power users who legitimately need elevated capabilities, grant them admin role or create specific service accounts.

## Auditing Policy Compliance

```bash
# Check for containers running with privileged mode
docker inspect $(docker ps -q) | \
  jq '[.[] | select(.HostConfig.Privileged == true) | {name: .Name, privileged: .HostConfig.Privileged}]'

# Check for host network mode containers
docker inspect $(docker ps -q) | \
  jq '[.[] | select(.HostConfig.NetworkMode == "host") | .Name]'

# Check for bind mounts
docker inspect $(docker ps -q) | \
  jq '[.[] | select(.HostConfig.Binds != null) | {name: .Name, binds: .HostConfig.Binds}]'
```

## Conclusion

Docker security policies in Portainer provide a centrally managed, enforceable security baseline for all container deployments. Implement these settings on day one and audit regularly to catch any containers that were deployed before policies were enabled.
