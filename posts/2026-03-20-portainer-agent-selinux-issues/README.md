# How to Fix SELinux Issues with Portainer Agent - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Agent, SELinux, RHEL, CentOS, Security

Description: Resolve SELinux permission denials that prevent the Portainer Agent from accessing the Docker socket or volumes on RHEL/CentOS systems.

## Introduction

On RHEL, CentOS, and other SELinux-enabled systems, SELinux may block the Portainer Agent from accessing the Docker socket or host volumes. These denials appear as silent failures - the container starts but cannot communicate with Docker. This guide covers diagnosing and fixing SELinux issues.

## Diagnosing SELinux Denials

```bash
# Check if SELinux is enabled and enforcing

getenforce
# If: Enforcing - SELinux is active and blocking
# If: Permissive - SELinux logs but doesn't block

# Check audit log for denials related to Docker/container
sudo ausearch -m avc -ts recent | grep -i "docker\|container\|portainer"

# Or check audit.log directly
sudo grep "avc:.*denied" /var/log/audit/audit.log | grep -i "docker\|container" | tail -20

# Check dmesg for SELinux messages
sudo dmesg | grep -i "avc\|selinux" | tail -20
```

## Fix 1: Use the :z Volume Mount Option

The `:z` option re-labels the volume content to allow container access:

```bash
# Add :z to allow Docker socket access
docker run -d \
  --name portainer_agent \
  --restart always \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock:z \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes:z \
  portainer/agent:latest
```

```yaml
# docker-compose.yml
services:
  agent:
    image: portainer/agent:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:z
      - /var/lib/docker/volumes:/var/lib/docker/volumes:z
```

The `:z` flag sets the SELinux label `svirt_sandbox_file_t` which allows container access.

## Fix 2: Use --privileged (Workaround, Not Recommended)

```bash
docker run -d \
  --name portainer_agent \
  --privileged \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/agent:latest
```

This disables SELinux confinement for the container entirely. Use only if other fixes fail.

## Fix 3: Set SELinux to Permissive Temporarily for Diagnosis

```bash
# Set permissive mode (temporarily, for testing)
sudo setenforce 0
getenforce  # Should show: Permissive

# If agent works in permissive mode, SELinux was the issue
# Re-enable enforcing
sudo setenforce 1
```

## Fix 4: Create a Custom SELinux Policy

The proper long-term fix is a custom SELinux policy:

```bash
# Generate a policy from the audit log denials
sudo ausearch -m avc -ts recent | audit2allow -M portainer-agent

# Review the generated policy
cat portainer-agent.te

# Install the policy
sudo semodule -i portainer-agent.pp

# Verify
sudo semodule -l | grep portainer
```

## Fix 5: Set the Docker Context

```bash
# Check current SELinux context of Docker socket
ls -Z /var/run/docker.sock
# Expected: system_u:object_r:container_var_run_t:s0 /var/run/docker.sock

# Restore default context if wrong
sudo restorecon -v /var/run/docker.sock

# Check container_var_run_t allows access
sesearch --allow --source container_t --target container_var_run_t --class sock_file
```

## Verifying the Fix

```bash
# Check agent container is running correctly
docker ps | grep portainer_agent

# Check logs for successful Docker connection
docker logs portainer_agent | grep -E "starting|connected|error" | head -10

# Test that agent responds
curl -s http://localhost:9001/ping
```

## Conclusion

SELinux denials are a common hurdle when deploying Portainer on enterprise Linux distributions. The `:z` volume mount option is the quickest fix that maintains SELinux protection. For a production system, create a proper SELinux policy module using `audit2allow` to specifically allow only what Portainer needs rather than using broad workarounds like `--privileged`.
