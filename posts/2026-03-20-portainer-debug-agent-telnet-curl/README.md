# How to Debug Agent Connectivity with Telnet and Curl

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Agent, Networking, Debugging

Description: Use telnet, curl, and netcat to systematically debug Portainer Agent connectivity issues from the network level up to the API level.

## Introduction

When the Portainer UI shows an agent as offline or unreachable, low-level network tools can quickly identify exactly where the connectivity breaks down. This guide teaches you to use `telnet`, `curl`, `nc` (netcat), and `openssl` to debug agent connectivity at each layer of the stack.

## Layer 1: Network Reachability (ICMP)

```bash
# Test basic IP connectivity to the agent host
ping -c 4 agent-host

# Expected: 0% packet loss, RTT < 50ms for LAN, < 200ms for WAN
# If ping fails:
# - Check routing: ip route show / route -n
# - Check if ICMP is blocked: some firewalls block ping but allow TCP
```

## Layer 2: TCP Port Connectivity (Netcat / Telnet)

```bash
# Test if the agent port is open and accepting connections
# Method 1: netcat (most reliable)
nc -zv agent-host 9001
# -z = scan only (don't send data)
# -v = verbose output

# Expected output:
# "Connection to agent-host 9001 port [tcp/*] succeeded!"

# Method 2: telnet
telnet agent-host 9001
# If connected: you'll see a blank screen or garbled data
# Type Ctrl+] then "quit" to exit

# Method 3: /dev/tcp (no tools required)
timeout 3 bash -c "cat < /dev/null > /dev/tcp/agent-host/9001" && echo "Open" || echo "Closed"
```

### Interpreting Results

| Result | Meaning |
|--------|---------|
| `Connection succeeded` | Port is open, agent is listening |
| `Connection refused` | Port is not open, agent not running |
| `No route to host` | Routing issue or host unreachable |
| `Connection timed out` | Firewall is blocking (silently dropping packets) |

## Layer 3: HTTP Protocol Test (Curl)

Once TCP connectivity is confirmed, test the HTTP layer:

```bash
# Test the agent ping endpoint
curl -v http://agent-host:9001/ping

# Expected response: some HTTP response (even 404 is OK — it means HTTP works)
# "Connection refused" = TCP level issue
# "Empty reply from server" = TCP connects but no HTTP response

# Test with timeout
curl -v --connect-timeout 5 --max-time 10 http://agent-host:9001/ping

# Test with all headers shown
curl -sI http://agent-host:9001/ping
```

## Layer 4: TLS Verification (OpenSSL)

If the agent uses TLS:

```bash
# Test TLS handshake
openssl s_client -connect agent-host:9001 -servername agent-host

# Check certificate details
openssl s_client -connect agent-host:9001 2>/dev/null | \
  openssl x509 -noout -text | grep -E "Subject:|Issuer:|Not After"

# Test with specific TLS version
openssl s_client -tls1_2 -connect agent-host:9001

# If TLS handshake fails:
# - Certificate is expired
# - CN/SAN doesn't match the hostname
# - TLS version mismatch
```

## Layer 5: Agent API Test

```bash
# Test the agent's API endpoint directly (if no auth required)
curl -v http://agent-host:9001/v1/browse/containers

# If the agent requires a secret, pass it as a header
curl -v -H "X-PortainerAgent-Signature: your-signature" \
  http://agent-host:9001/v1/docker/containers/json

# Check what the agent API exposes
curl -s http://agent-host:9001 | head -50
```

## Debug from Inside the Portainer Container

```bash
# Portainer server debugging — test connectivity from inside the Portainer container
docker exec -it portainer sh

# Inside the container:
# Test DNS resolution of agent
nslookup agent-host

# Test connectivity
wget -qO- http://agent-host:9001/ping
# or
curl http://agent-host:9001/ping

# Test if agent is reachable from Portainer's network
ping agent-host

exit
```

## Debug Firewall Rules with tcpdump

```bash
# On the Portainer server, capture traffic to/from the agent
sudo tcpdump -i any host agent-host and port 9001 -n

# In another terminal, trigger an action in Portainer that contacts the agent
# Then observe the tcpdump output:
# SYN sent, SYN-ACK received = connection establishing
# SYN sent, RST received = port refused
# SYN sent, no response = firewalled/dropped

# On the agent host, capture to see if packets arrive
sudo tcpdump -i any port 9001 -n
```

## Trace the Full Path with Traceroute

```bash
# Find where packets are being dropped
traceroute agent-host

# For more detail with TCP instead of UDP
sudo traceroute -T -p 9001 agent-host

# For ICMP traceroute
sudo traceroute -I agent-host
```

## Quick Debug Script

```bash
#!/bin/bash
# portainer-agent-debug.sh
AGENT_HOST="${1:-agent-host}"
AGENT_PORT="${2:-9001}"

echo "=== Layer 1: ICMP Ping ==="
ping -c 3 "$AGENT_HOST" 2>&1 | tail -3

echo "=== Layer 2: TCP Connectivity ==="
nc -zv "$AGENT_HOST" "$AGENT_PORT" 2>&1

echo "=== Layer 3: HTTP Response ==="
curl -s --connect-timeout 5 "http://$AGENT_HOST:$AGENT_PORT" -o /dev/null -w "HTTP Status: %{http_code}\n"

echo "=== Agent Container Status (if local) ==="
docker ps | grep portainer-agent 2>/dev/null || echo "Agent not local"

echo "Done"
```

Run with:
```bash
chmod +x portainer-agent-debug.sh
./portainer-agent-debug.sh my-agent-host 9001
```

## Conclusion

Debugging Portainer Agent connectivity is a structured process: start at the network layer with ping to verify routing, then confirm the TCP port is open with `nc -zv`, then test the HTTP layer with `curl -v`. If all three work but Portainer still can't connect, the issue is authentication (agent secret mismatch) or TLS configuration. Use `openssl s_client` to debug TLS and check the agent secret in both the agent environment variables and the Portainer environment settings.
