# How to Troubleshoot Rancher Agent Connection Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, Agent

Description: Learn how to diagnose and fix Rancher agent connection problems, including network issues, certificate mismatches, and proxy configuration.

## Introduction

The `cattle-cluster-agent` and `cattle-node-agent` are responsible for maintaining the connection between downstream clusters and the Rancher management server. When these agents lose connectivity, clusters appear as "Unavailable" in the Rancher UI, and operations like deploying workloads stop working. This guide covers how to systematically restore connectivity.

## Understanding the Agent Architecture

```text
Rancher Server (management cluster)
        ↑  WebSocket connection (wss://)
cattle-cluster-agent (downstream cluster)
        ↑  SSH tunnel or direct
cattle-node-agent (each node in downstream cluster)
```

The `cattle-cluster-agent` establishes an outbound WebSocket tunnel to Rancher. All cluster API traffic flows through this tunnel.

## Step 1: Check Agent Pod Status

```bash
# Check the cluster agent

kubectl get pods -n cattle-system

# Expected: cattle-cluster-agent-<hash>   1/1   Running
# Problem:  cattle-cluster-agent-<hash>   0/1   CrashLoopBackOff

# Get detailed events
kubectl describe pod -n cattle-system -l app=cattle-cluster-agent

# Stream agent logs
kubectl logs -n cattle-system -l app=cattle-cluster-agent -f --tail=200
```

## Step 2: Check Network Connectivity

The agent must reach the Rancher server URL on port 443:

```bash
# Test from inside the cluster using a debug pod
kubectl run net-debug --rm -it \
  --image=nicolaka/netshoot \
  --restart=Never \
  -- curl -v https://<rancher-url>/healthz

# Check DNS resolution
kubectl run dns-debug --rm -it \
  --image=nicolaka/netshoot \
  --restart=Never \
  -- nslookup <rancher-hostname>

# Test WebSocket connectivity
kubectl run ws-debug --rm -it \
  --image=nicolaka/netshoot \
  --restart=Never \
  -- curl -v --no-buffer \
     -H "Connection: Upgrade" \
     -H "Upgrade: websocket" \
     https://<rancher-url>/k8s/clusters/local
```

## Step 3: Verify the Cattle Server URL

The agent uses the `CATTLE_SERVER` environment variable to know where to connect:

```bash
# Check the current server URL setting
kubectl get setting -n cattle-system server-url -o jsonpath='{.value}'
# or via Rancher API
curl -sk -H "Authorization: Bearer $TOKEN" \
  https://<rancher-url>/v3/settings/server-url | jq .value
```

If the URL is wrong, update it:

```bash
# Update via the Rancher UI: Global Settings → Server URL
# Or via API:
curl -sk -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "https://rancher.correct-url.com"}' \
  "https://<rancher-url>/v3/settings/server-url"
```

## Step 4: Check Certificate Trust

If Rancher uses a private or self-signed CA, the agent must trust it:

```bash
# Check the cacerts setting
kubectl get secret -n cattle-system cattle-ca -o jsonpath='{.data.cacerts}' \
  | base64 -d | openssl x509 -noout -subject -issuer -dates

# Agent logs showing TLS errors
# "x509: certificate signed by unknown authority"
# → The agent doesn't trust Rancher's CA

# Re-apply the CA cert to the agent
kubectl patch deployment cattle-cluster-agent -n cattle-system \
  --patch '{"spec":{"template":{"spec":{"containers":[{
    "name":"cluster-register",
    "env":[{"name":"CATTLE_CA_CHECKSUM","value":"<new-ca-checksum>"}]
  }]}}}}'
```

## Step 5: Check Proxy Configuration

If your cluster nodes use an HTTP proxy, the agent needs matching configuration:

```bash
# Check existing proxy env vars on the agent
kubectl get deployment -n cattle-system cattle-cluster-agent -o json \
  | jq '.spec.template.spec.containers[].env[] | select(.name | contains("PROXY"))'

# The agent respects standard proxy env vars:
# HTTP_PROXY, HTTPS_PROXY, NO_PROXY
# Ensure <rancher-url> is in NO_PROXY if the proxy is in the same network
```

Update proxy settings:

```bash
kubectl set env deployment/cattle-cluster-agent -n cattle-system \
  HTTPS_PROXY=http://proxy.example.com:3128 \
  NO_PROXY=10.0.0.0/8,localhost,<rancher-internal-ip>
```

## Step 6: Force Re-registration

If all else fails, force the agent to re-register with Rancher:

```bash
# Delete the agent deployment - Rancher will recreate it automatically
kubectl delete deployment -n cattle-system cattle-cluster-agent

# Watch the agent come back
kubectl get pods -n cattle-system -w
```

Alternatively, from the Rancher UI:
1. Navigate to **Cluster Management** → select the affected cluster.
2. Click **⋮ → Edit Config**.
3. Scroll down and click **Registration Command**.
4. Re-run the generated `kubectl apply` command on the downstream cluster.

## Step 7: Check Firewall and Security Groups

Ensure the downstream cluster's egress rules allow:

| Protocol | Port | Destination |
|---|---|---|
| TCP | 443 | Rancher server |
| TCP | 443 | Container registry (if pulling agent image) |

## Conclusion

Rancher agent connection issues almost always trace back to network reachability, TLS certificate trust, incorrect server URLs, or proxy misconfiguration. Work through each layer methodically - pod status, network connectivity, certificate validity, and proxy settings - and the agent will re-establish its connection to the Rancher management server.
