# How to Rotate K3s Token

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Security, Tokens, DevOps

Description: Learn how to rotate the K3s cluster token to maintain security and prevent unauthorized nodes from joining your cluster.

## Introduction

The K3s cluster token (`K3S_TOKEN`) is a shared secret used to authenticate nodes joining the cluster. If this token is compromised, an attacker could join unauthorized nodes to your cluster. Rotating the token periodically — or immediately after a suspected compromise — is a critical security practice. This guide explains the process for safely rotating the K3s token without disrupting running workloads.

## Understanding K3s Token Usage

The K3s token serves multiple purposes:
- **Node authentication**: Agents use it to authenticate when joining the server
- **Certificate bootstrapping**: Used to bootstrap kubelet client certificates
- **Data encryption**: Used as part of the key derivation for cluster secrets

The token is stored at `/var/lib/rancher/k3s/server/node-token` on the server and referenced by agents during startup.

## Prerequisites

- Root/sudo access to all K3s nodes
- A maintenance window (service restarts required)
- List of all agent nodes in the cluster

## Step 1: Check the Current Token

```bash
# View the current node token
cat /var/lib/rancher/k3s/server/node-token

# The token has the format:
# K10<hash>::server:<secret>
# or simply a plain secret if set manually
```

## Step 2: Generate a New Token

You can use any sufficiently random string as the new token:

```bash
# Generate a cryptographically secure random token
NEW_TOKEN=$(openssl rand -hex 32)
echo "New token: $NEW_TOKEN"

# Store the new token for reference
echo "$NEW_TOKEN" > /tmp/new-k3s-token.txt
```

## Step 3: Update the Server Token

### Option A: Using K3s Config File

```bash
# Stop K3s server
systemctl stop k3s

# Update the token in the K3s config
# /etc/rancher/k3s/config.yaml
cat > /etc/rancher/k3s/config.yaml << EOF
token: "your-new-secure-token-here"
# ... other existing config options
EOF

# Update the token file directly
echo "your-new-secure-token-here" > /var/lib/rancher/k3s/server/node-token

# Start K3s server
systemctl start k3s

# Verify the server is healthy
kubectl get nodes
```

### Option B: Using Environment Variables

```bash
# Stop K3s
systemctl stop k3s

# Update the environment file
cat > /etc/systemd/system/k3s.service.env << EOF
K3S_TOKEN=your-new-secure-token-here
EOF

# Reload systemd
systemctl daemon-reload

# Start K3s
systemctl start k3s
```

## Step 4: Update Agent Nodes

Every agent node must be updated with the new token. Agents will fail to communicate with the server if their token doesn't match:

```bash
#!/bin/bash
# update-agent-token.sh
# Run this on each agent node

NEW_TOKEN="your-new-secure-token-here"

# Stop the agent service
systemctl stop k3s-agent

# Update the token in the agent environment file
if [ -f /etc/systemd/system/k3s-agent.service.env ]; then
  # Update existing env file
  sed -i "s/K3S_TOKEN=.*/K3S_TOKEN=$NEW_TOKEN/" \
    /etc/systemd/system/k3s-agent.service.env
else
  # Create env file
  echo "K3S_TOKEN=$NEW_TOKEN" > /etc/systemd/system/k3s-agent.service.env
fi

# Reload systemd
systemctl daemon-reload

# Start the agent
systemctl start k3s-agent

echo "Agent token updated and service restarted"
```

If agents were installed with the token embedded in `/etc/rancher/k3s/config.yaml`:

```bash
# Stop agent
systemctl stop k3s-agent

# Update config
cat > /etc/rancher/k3s/config.yaml << EOF
server: "https://<server-ip>:6443"
token: "your-new-secure-token-here"
EOF

# Restart agent
systemctl start k3s-agent
```

## Step 5: Verify All Nodes Have Reconnected

After updating all agents, verify the cluster is healthy:

```bash
# Check all nodes are Ready
kubectl get nodes

# If an agent hasn't reconnected, check its logs
journalctl -u k3s-agent -f

# Common error if token is wrong:
# "Failed to connect to server: dial tcp ... connection refused"
# or
# "Node password rejected"
```

## Step 6: Update Agent Password Files

K3s agents store a per-node password that is validated against the server. After token rotation, you may need to clear stale password files:

```bash
# On the K3s server, remove old node password entries
# (allows nodes to re-authenticate with the new token)
rm -f /var/lib/rancher/k3s/server/cred/node-passwd

# Alternatively, remove the specific node entry
ls /var/lib/rancher/k3s/server/cred/
```

## Automating Token Rotation

For environments requiring regular token rotation, create a rotation script:

```bash
#!/bin/bash
# /usr/local/bin/rotate-k3s-token.sh

set -euo pipefail

AGENT_NODES=("192.168.1.11" "192.168.1.12" "192.168.1.13")
SSH_USER="root"

# Generate new token
NEW_TOKEN=$(openssl rand -hex 32)
echo "Generated new token: ${NEW_TOKEN:0:8}..."

# Update server
systemctl stop k3s
echo "$NEW_TOKEN" > /var/lib/rancher/k3s/server/node-token
systemctl start k3s
echo "Server updated"

# Update each agent via SSH
for NODE in "${AGENT_NODES[@]}"; do
  echo "Updating agent: $NODE"
  ssh "$SSH_USER@$NODE" "
    systemctl stop k3s-agent
    echo 'K3S_TOKEN=$NEW_TOKEN' > /etc/systemd/system/k3s-agent.service.env
    systemctl daemon-reload
    systemctl start k3s-agent
  "
  echo "Agent $NODE updated"
done

echo "Token rotation complete"
```

## Conclusion

Rotating the K3s token is a straightforward but important security practice. The process requires coordinating updates across all server and agent nodes with brief service restarts. For multi-agent clusters, consider using configuration management tools like Ansible to automate the token update across all nodes simultaneously, minimizing the maintenance window duration.
