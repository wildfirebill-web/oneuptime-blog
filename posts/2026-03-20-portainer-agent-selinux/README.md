# How to Fix Agent Issues When SELinux Is Enabled

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, SELinux, Security, Agent

Description: Resolve Portainer Agent connection and permission failures caused by SELinux enforcing policies on RHEL, CentOS, and Fedora systems.

## Introduction

SELinux (Security-Enhanced Linux) enforces mandatory access control policies that can prevent the Portainer Agent from accessing the Docker socket, volume paths, and network ports — even when standard Linux permissions would allow it. This guide covers how to diagnose SELinux-related Agent issues and apply the appropriate fixes without completely disabling SELinux.

## Step 1: Verify SELinux Is the Cause

```bash
# Check SELinux status
getenforce
# Outputs: Enforcing, Permissive, or Disabled

sestatus
# Shows detailed SELinux configuration

# Quick test: temporarily set to Permissive and see if issue resolves
sudo setenforce 0
docker restart portainer-agent

# Test connectivity from Portainer server
# If it works in Permissive mode, SELinux is the cause
# Re-enable Enforcing after testing
sudo setenforce 1
```

## Step 2: Check SELinux Audit Logs

```bash
# Check for AVC (Access Vector Cache) denial messages
sudo ausearch -m AVC -ts recent

# Or check the audit log directly
sudo cat /var/log/audit/audit.log | grep portainer | tail -20

# Filter for denied actions specifically
sudo ausearch -m AVC -c docker --raw | grep DENIED

# Get a summary of all denials
sudo aureport --avc
```

## Step 3: Fix Docker Socket Access

The most common SELinux issue is the agent being denied access to `/var/run/docker.sock`:

```bash
# The :z label option tells Docker to relabel the socket for the container
docker stop portainer-agent && docker rm portainer-agent

docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock:z \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes:z \
  portainer/agent:latest

# :z = shared relabeling (other containers can also access)
# :Z = private relabeling (exclusive access for this container)
```

## Step 4: Fix Volume Path Access

If the agent can't access Docker volume paths:

```bash
# Check the SELinux context on Docker volumes directory
ls -lZ /var/lib/docker/volumes/

# Set the correct SELinux context
sudo chcon -Rt svirt_sandbox_file_t /var/lib/docker/volumes/

# Alternatively, use the :Z mount option
docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock:z \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes:Z \
  portainer/agent:latest
```

## Step 5: Create a Custom SELinux Policy Module

For a permanent, precise fix:

```bash
# Install SELinux tools
sudo dnf install -y policycoreutils-python-utils selinux-policy-devel

# Generate a policy module from denial messages
sudo ausearch -m AVC -ts recent --raw | audit2allow -M portainer-agent

# Review the generated policy
cat portainer-agent.te

# Install the policy module
sudo semodule -i portainer-agent.pp

# Verify installation
sudo semodule -l | grep portainer
```

## Step 6: Allow Network Port Access

If the agent port (9001) is blocked by SELinux:

```bash
# Check if port 9001 has an SELinux context
sudo semanage port -l | grep 9001

# Add the port to the allowed list for container use
sudo semanage port -a -t http_port_t -p tcp 9001

# Or add to container ports
sudo semanage port -a -t container_port_t -p tcp 9001

# Verify
sudo semanage port -l | grep 9001
```

## Step 7: Enable Container_manage_cgroup Boolean

On RHEL/CentOS, enable the SELinux boolean that allows containers to manage cgroups:

```bash
# Check current boolean states
sudo getsebool -a | grep container

# Enable key booleans for Docker/container access
sudo setsebool -P container_manage_cgroup true
sudo setsebool -P container_use_cephfs true  # If using Ceph storage

# For network access
sudo setsebool -P container_connect_any true  # Allow containers to connect to any port

# Make changes permanent (-P flag)
sudo setsebool -P domain_kernel_load_modules true
```

## Step 8: Use the Privileged Flag (Last Resort)

If precise SELinux policy fixing is not feasible:

```bash
# Run agent with --privileged (bypasses SELinux type enforcement for the container)
# WARNING: This reduces container isolation
docker run -d \
  -p 9001:9001 \
  --name portainer-agent \
  --restart=unless-stopped \
  --privileged \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

## Step 9: Check and Fix SELinux Context on Agent Data Volume

```bash
# Check the SELinux context of the agent data directory
ls -lZ /var/lib/docker/volumes/portainer_agent_data/_data/

# Set correct context for Docker volume data
sudo chcon -Rt container_file_t \
  /var/lib/docker/volumes/portainer_agent_data/_data/
```

## Step 10: Permanent Solution with Dockerfile Label

For the most SELinux-compatible deployment, use Docker Compose with explicit SELinux labels:

```yaml
services:
  portainer-agent:
    image: portainer/agent:latest
    ports:
      - "9001:9001"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:z
      - /var/lib/docker/volumes:/var/lib/docker/volumes:z
    restart: unless-stopped
    security_opt:
      - label:disable  # Disable SELinux label enforcement for this container
      # Or use a specific label:
      # - label:type:container_runtime_t
```

## Conclusion

SELinux issues with the Portainer Agent are best fixed by using the `:z` or `:Z` volume mount label, which tells Docker to relabel the mounted paths for SELinux access. For network port issues, use `semanage port` to add port 9001 to the allowed list. Avoid disabling SELinux entirely — instead, use `audit2allow` to generate a precise policy module that allows only what Portainer needs.
