# How to Configure IPv6 ExternalIPs in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv6, Networking, ExternalIPs, Services

Description: A practical guide to configuring IPv6 ExternalIPs on Kubernetes Services to expose workloads directly via IPv6 addresses.

Kubernetes Services can be assigned `externalIPs` that route external traffic to the service. With IPv6, this means you can expose a service directly on a publicly routable IPv6 address assigned to one of your cluster nodes.

## Prerequisites

- A Kubernetes cluster with IPv6 support enabled (dual-stack or IPv6-only)
- Nodes with IPv6 addresses assigned on their network interfaces
- `kubectl` access with cluster-admin rights

## Understanding ExternalIPs

The `externalIPs` field in a Service spec tells kube-proxy to intercept traffic destined for those addresses and route it to the service's endpoints. Unlike LoadBalancer services, no cloud controller is needed - the IP must be manually assigned to a node.

## Step 1: Verify Node IPv6 Addresses

Before configuring externalIPs, confirm which IPv6 addresses are assigned to your nodes.

```bash
# List IPv6 addresses on all nodes

kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[*].address}{"\n"}{end}'
```

On the node itself:

```bash
# Show all IPv6 addresses on the node
ip -6 addr show
```

## Step 2: Create a Service with IPv6 ExternalIPs

The following manifest creates a ClusterIP service with an IPv6 externalIP pointing to a node address.

```yaml
# service-ipv6-externalip.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-ipv6-service
  namespace: default
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  # Specify the IPv6 address assigned to one or more cluster nodes
  externalIPs:
    - "2001:db8::10"
```

Apply it:

```bash
kubectl apply -f service-ipv6-externalip.yaml
```

## Step 3: Verify the Service

```bash
# Confirm the external IP appears in the service spec
kubectl get svc my-ipv6-service -o wide
```

Expected output will show `EXTERNAL-IP` as `2001:db8::10`.

## Step 4: Test Connectivity

From an external host with IPv6 connectivity:

```bash
# Use curl with IPv6 address notation (brackets required)
curl -6 http://[2001:db8::10]:80/
```

## Step 5: Configure iptables / kube-proxy Verification

Kube-proxy creates iptables or IPVS rules for externalIPs. Verify these rules are present on the node:

```bash
# Check ip6tables rules for the external IP
sudo ip6tables -t nat -L KUBE-SERVICES -n | grep "2001:db8::10"
```

## Important Considerations

- **Security**: Any host that can route to that IPv6 address can reach the service. Ensure network policies restrict unwanted access.
- **Node assignment**: The IPv6 address in `externalIPs` must actually be assigned to a node's interface or the traffic will be dropped.
- **Multiple IPs**: You can list multiple IPv6 (or mixed IPv4/IPv6) addresses in the `externalIPs` array for dual-stack exposure.

## Monitoring with OneUptime

Once your service is exposed via an IPv6 externalIP, set up an [OneUptime](https://oneuptime.com) monitor targeting the IPv6 endpoint to track uptime and latency from multiple global vantage points.

```bash
# Quick connectivity check from a monitoring perspective
ping6 2001:db8::10
```

Configuring IPv6 ExternalIPs is a lightweight way to expose Kubernetes services without a cloud load balancer, making it ideal for bare-metal and on-premises deployments.
