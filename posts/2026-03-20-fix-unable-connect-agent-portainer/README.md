# How to Fix 'Unable to Connect to Agent' Errors in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Agent, Docker, Networking, Connectivity

Description: Learn how to diagnose and fix 'Unable to Connect to Agent' errors in Portainer by checking network connectivity, TLS certificates, and agent secret configuration.

---

The "Unable to Connect to Agent" error appears in Portainer when the server cannot reach the Portainer Agent on a remote host. The causes range from firewall rules to TLS mismatches to wrong port configurations.

## Understand the Connection Flow

```mermaid
graph LR
    A[Portainer Server] -->|TCP 9001| B[Portainer Agent]
    B -->|Docker Socket| C[Docker Daemon]
```

The server initiates a TCP connection to the agent on port 9001. Both firewall and application-level issues can break this.

## Step 1: Verify Agent is Running

On the agent host:

```bash
# Check if the agent container is running

docker ps | grep portainer_agent

# If not running, check for startup errors
docker logs portainer_agent --tail 50
```

## Step 2: Test Network Connectivity

From the Portainer server host:

```bash
# Test TCP connectivity to the agent port
telnet <agent-host-ip> 9001

# Or using nc (netcat)
nc -zv <agent-host-ip> 9001

# Expected: "Connection succeeded"
# If connection refused: firewall or agent not listening
# If timeout: network path blocked
```

## Step 3: Check Firewall Rules

On the agent host, ensure port 9001 is open:

```bash
# UFW
sudo ufw allow 9001/tcp

# iptables
sudo iptables -A INPUT -p tcp --dport 9001 -j ACCEPT

# firewalld
sudo firewall-cmd --permanent --add-port=9001/tcp
sudo firewall-cmd --reload
```

## Step 4: Verify Agent Secret Match

Both the server and agent must use the same secret:

```bash
# Portainer server must be started with:
--agent-secret mysecrettoken

# Portainer Agent must be started with:
-e AGENT_SECRET=mysecrettoken
```

If the secrets differ, the TLS handshake succeeds but authentication fails with "Unable to Connect."

## Step 5: Confirm Agent is Listening on 0.0.0.0

The agent must bind to all interfaces, not just localhost:

```bash
# Check what address the agent is listening on
docker exec portainer_agent netstat -tlnp | grep 9001
# Should show: 0.0.0.0:9001
```

## Step 6: Re-add the Environment in Portainer

If network and secrets are correct, remove and re-add the environment in Portainer:

1. Go to **Environments > Select the environment > Remove**.
2. Go to **Environments > Add Environment > Docker Agent**.
3. Enter the correct IP and port `9001`.
4. Click **Connect**.
