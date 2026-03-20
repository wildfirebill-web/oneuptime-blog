# How to Configure Kubernetes Services with externalIPs for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Services, ExternalIPs, IPv4, Networking, Load Balancing

Description: Use Kubernetes Service externalIPs to bind a Service to one or more specific external IPv4 addresses, enabling traffic routing to cluster pods from outside the cluster.

## Introduction

Kubernetes Services normally expose applications through NodePort, LoadBalancer, or ClusterIP. The `externalIPs` field is an alternative that binds the Service to one or more external IPv4 addresses assigned to cluster nodes. Traffic arriving on a node at the specified external IP and port is routed to the Service's pods.

## When to Use externalIPs

- **Bare-metal clusters** without a cloud load balancer controller
- Binding a Service to a specific VIP (Virtual IP) managed externally (e.g., keepalived)
- Providing a predictable external IP without a LoadBalancer service type

## Basic externalIPs Service

```yaml
# service-externalips.yaml

apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: default
spec:
  selector:
    app: web               # Selects pods with this label
  type: ClusterIP          # Can also use NodePort or LoadBalancer
  ports:
    - name: http
      port: 80             # Service port
      targetPort: 8080     # Pod port
      protocol: TCP
  externalIPs:
    - 192.168.1.200        # External IPv4 address assigned to a cluster node
    - 192.168.1.201        # Additional external IP (optional)
```

Apply the manifest:

```bash
kubectl apply -f service-externalips.yaml

# Verify the Service
kubectl get service web-service
```

## How It Works

The kube-proxy on each node installs iptables rules for the externalIPs. When a packet arrives at a node destined for `192.168.1.200:80`, kube-proxy intercepts it and load-balances it to one of the matching pods, regardless of which node the pod runs on.

## Assigning the External IP to a Node

The external IP must be assigned to a network interface on at least one cluster node. On bare-metal:

```bash
# Add the virtual IP to a node's interface
sudo ip addr add 192.168.1.200/24 dev eth0

# For persistence across reboots (with Netplan)
# Add 192.168.1.200/24 to the interface's addresses list
```

Using keepalived for HA (highly recommended for production):

```bash
# keepalived moves the VIP to the active node automatically
# Configure keepalived to manage 192.168.1.200
```

## Verifying Traffic Routing

```bash
# From outside the cluster, test the external IP
curl http://192.168.1.200/

# Check iptables rules kube-proxy created
sudo iptables -t nat -L -n | grep 192.168.1.200
```

## Security Consideration

**externalIPs are not validated** by Kubernetes. A user with permission to create Services could specify any IP, potentially redirecting traffic from legitimate external services. Restrict who can create Services with externalIPs using admission webhooks or RBAC.

```yaml
# Example OPA/Gatekeeper constraint to block unauthorized externalIPs
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoExternalIPs
metadata:
  name: no-external-ips
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Service"]
```

## MetalLB as an Alternative

For production bare-metal clusters, MetalLB manages external IPs automatically using BGP or ARP announcements:

```bash
# MetalLB is generally preferred over manual externalIPs for production
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

## Conclusion

`externalIPs` provides a simple way to expose Kubernetes Services on specific IPv4 addresses without a cloud load balancer. For production bare-metal deployments, combine it with keepalived for VIP failover or migrate to MetalLB for automated IP management.
