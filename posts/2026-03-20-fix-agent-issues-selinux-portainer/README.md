# How to Fix Agent Issues When SELinux Is Enabled

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, SELinux, Docker, Security, CentOS, RHEL

Description: Learn how to fix Portainer Agent failures caused by SELinux enforcement on RHEL, CentOS, and Fedora systems by applying correct SELinux labels and policies.

---

SELinux (Security-Enhanced Linux) is enabled by default on RHEL, CentOS, and Fedora. It uses mandatory access controls that can prevent the Portainer Agent from accessing the Docker socket or host volumes, even when the container has the correct Unix permissions.

## Symptoms of SELinux Interference

```bash
# Agent container starts but Portainer shows "Unable to Connect"
docker logs portainer_agent 2>&1 | grep -i "permission\|denied\|selinux"

# Check for SELinux denials in the audit log
sudo ausearch -c docker -m avc --start today

# Quick check for any SELinux denials related to Docker
sudo sealert -a /var/log/audit/audit.log | grep docker
```

## Option 1: Use the :Z or :z Volume Label

The most targeted fix is to add SELinux labels to volume mounts. The `:Z` label sets a private unshared label (for one container) and `:z` sets a shared label:

```bash
# Redeploy the agent with :Z labels on socket and volume mounts
docker run -d \
  --name portainer_agent \
  --restart=always \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock:Z \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes:Z \
  portainer/agent:latest
```

## Option 2: Apply a Custom SELinux Policy

Generate and apply a policy from the audit denials:

```bash
# Capture denials while the agent is running and failing
sudo ausearch -c docker -m avc --start today | \
  audit2allow -M portainer_agent

# Review the generated policy
cat portainer_agent.te

# Apply the policy module
sudo semodule -i portainer_agent.pp
```

## Option 3: Set SELinux to Permissive Mode (Testing Only)

To confirm SELinux is the cause, temporarily set it to permissive:

```bash
# Set permissive mode (does not enforce, but logs denials)
sudo setenforce 0

# If Portainer Agent now works, SELinux was the cause
# Re-enable enforcing and apply a proper policy instead
sudo setenforce 1
```

Do not leave SELinux in permissive mode on production systems.

## Option 4: Label the Docker Socket Context

```bash
# Check current context of the Docker socket
ls -lZ /var/run/docker.sock

# Relabel it with the correct context for container access
sudo chcon -t container_file_t /var/run/docker.sock
```

Restart the agent after any context changes and verify with `docker logs portainer_agent`.
