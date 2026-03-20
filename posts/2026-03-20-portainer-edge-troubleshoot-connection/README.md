# How to Troubleshoot Edge Agent Connection Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Edge Agent, Troubleshooting, Connectivity, Debugging

Description: Diagnose and fix common Edge Agent connectivity failures in Portainer including tunnel server issues, edge key problems, and network errors.

## Introduction

Edge Agent connections fail for different reasons than standard agent connections. Since edge agents initiate outbound connections, the issues often involve outbound firewall rules, the edge key validity, or tunnel server accessibility rather than inbound port availability.

## Step 1: Check Agent Logs

```bash
# View recent logs
docker logs portainer_edge_agent --tail 50

# Common successful log entries:
# "Starting Portainer agent"
# "Edge agent mode enabled"
# "Connecting to tunnel server"
# "Tunnel established"

# Common error messages:
# "Failed to connect to tunnel server: dial tcp..."
# "Invalid edge key"
# "TLS certificate verification failed"
```

## Step 2: Verify the Edge Key

The edge key contains the Portainer server URL and authentication credentials. Decode it:

```bash
# Edge key is base64-encoded
EDGE_KEY="your-edge-key-here"
echo $EDGE_KEY | base64 -d | python3 -m json.tool

# Expected format:
# {
#   "portainerInstanceID": "...",
#   "portainerURL": "https://portainer.example.com",
#   "tunnelServerAddr": "portainer.example.com:8000",
#   "credentials": "..."
# }
```

Verify:
- `portainerURL` is the correct Portainer HTTPS URL
- `tunnelServerAddr` is accessible from the agent host

## Step 3: Test Tunnel Server Connectivity

```bash
# Test connectivity to the tunnel server (port 8000)
nc -zv portainer.example.com 8000

# From inside the agent container
docker exec portainer_edge_agent nc -zv portainer.example.com 8000
```

If port 8000 is blocked, this is the most common cause.

## Step 4: Check Outbound Firewall Rules

```bash
# On the agent host
# Test outbound to Portainer tunnel server
curl -v --connect-timeout 10 http://portainer.example.com:8000/

# Check iptables for outbound blocks
sudo iptables -L OUTPUT -n | grep -E "8000|portainer"

# Check if DNS resolves
nslookup portainer.example.com
```

## Step 5: Verify HTTPS Access to Portainer

```bash
# Test HTTPS access to Portainer API
curl -sk https://portainer.example.com/api/system/version
# Should return version information

# If this fails, the agent can't register
```

## Step 6: Regenerate the Edge Key

If the edge key is invalid or expired:

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Get edge environments
curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/endpoints \
  | python3 -c "
import sys, json
for e in json.load(sys.stdin):
    if e.get('Type') in [4, 7, 8]:
        print(f'Edge Env ID={e[\"Id\"]} Name={e[\"Name\"]} Key={e.get(\"EdgeKey\",\"\")}')
"
```

Update the agent with the new key:
```bash
docker stop portainer_edge_agent && docker rm portainer_edge_agent

docker run -d \
  --name portainer_edge_agent \
  -e EDGE=1 \
  -e EDGE_ID=device-id \
  -e EDGE_KEY=new-edge-key \
  portainer/agent:latest
```

## Common Error Table

| Error | Cause | Fix |
|-------|-------|-----|
| `connection refused :8000` | Port 8000 blocked | Open outbound port 8000 |
| `Invalid edge key` | Key corrupted or wrong | Regenerate key |
| `TLS certificate verify failed` | Self-signed cert | Add `EDGE_INSECURE_POLL=1` |
| `context deadline exceeded` | Network timeout | Check connectivity, increase timeout |
| Environment shows "Down" | Agent not checking in | Check agent is running |

## Conclusion

Edge Agent connectivity issues almost always involve network access to port 8000 (tunnel server), edge key validity, or TLS certificate trust. The diagnostic path: check logs → decode and verify edge key → test port 8000 outbound → verify HTTPS access → regenerate key if needed. Start with the simplest checks before investigating complex network routing issues.
