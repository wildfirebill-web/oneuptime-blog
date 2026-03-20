# How to Fix Edge Agent Not Connecting to Portainer Server

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Edge Agent, Connectivity, Firewall, TLS

Description: Learn how to diagnose and fix Portainer Edge Agent connection failures, including tunnel port issues, key mismatches, and firewall configuration.

---

The Portainer Edge Agent reverses the usual connection direction: it dials out from the remote site to the Portainer server. This makes it ideal for networks without inbound port forwarding, but it requires the server to be publicly reachable on the tunnel port.

## How Edge Agent Connections Work

```mermaid
graph LR
    A[Edge Agent] -->|Outbound TCP 8000| B[Portainer Server tunnel port]
    B --> C[Portainer API]
```

The edge agent connects outbound to the Portainer server's tunnel port (default 8000). The server must have this port publicly accessible.

## Step 1: Verify the Edge Key

The edge key encodes the Portainer server URL and tunnel port. It is generated when you create a new Edge environment in Portainer. A wrong key causes silent connection failures.

```bash
# On the edge host, verify the agent container started with the correct key

docker logs portainer_edge_agent 2>&1 | head -20

# Look for:
# level=info msg="Edge key decoded successfully"
# If you see "Failed to decode edge key", the key is wrong or corrupt
```

## Step 2: Test Tunnel Port Accessibility

From the edge site, verify the Portainer server's tunnel port is reachable:

```bash
# Test connectivity to the Portainer tunnel port
curl -v telnet://portainer-server.example.com:8000

# Or with nc
nc -zv portainer-server.example.com 8000
```

If this fails, open port 8000 on the Portainer server's firewall.

## Step 3: Ensure Portainer Server Has Tunnel Port Exposed

When running Portainer, the tunnel port must be published:

```bash
docker run -d \
  --name portainer \
  -p 9000:9000 \
  -p 9443:9443 \
  -p 8000:8000 \    # REQUIRED for Edge Agent connections
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Step 4: Re-generate the Edge Key

If the key is suspect, delete the edge environment in Portainer, re-create it, and use the new deployment command with the fresh key on the edge host.

## Step 5: Check for Proxy Interference

If the edge site routes traffic through an HTTP proxy, the edge agent needs proxy settings:

```bash
docker run -d \
  -e HTTPS_PROXY=http://proxy.corp.example.com:3128 \
  -e NO_PROXY=localhost,127.0.0.1 \
  portainer/agent:latest
```

## Step 6: Verify Agent Logs After Fix

```bash
# After correcting the configuration, watch for successful connection
docker logs -f portainer_edge_agent

# Successful connection looks like:
# level=info msg="Connecting to edge server..."
# level=info msg="Connection to Portainer server established"
```
