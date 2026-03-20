# How to Expose Portainer on Kubernetes via LoadBalancer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, LoadBalancer, Networking, DevOps

Description: Learn how to expose Portainer on Kubernetes using a LoadBalancer service for production-ready external access.

## Introduction

Using a LoadBalancer service to expose Portainer on Kubernetes provides a stable, cloud-managed external IP address. Cloud providers (AWS, Azure, GCP) automatically provision a load balancer and assign a public IP. On-premises clusters can use MetalLB for the same functionality. This guide covers LoadBalancer configuration for Portainer.

## Prerequisites

- Kubernetes cluster on a cloud provider (or MetalLB on-premises)
- Portainer installed via Helm
- Cluster-admin access

## Step 1: Install Portainer with LoadBalancer Service

```bash
helm install portainer portainer/portainer \
  --namespace portainer \
  --set service.type=LoadBalancer

# Or with custom values:

```

```yaml
# portainer-lb-values.yaml
service:
  type: LoadBalancer
  # Optionally specify a static IP (cloud-provider dependent):
  # loadBalancerIP: "203.0.113.100"
  annotations:
    # AWS: use a Network Load Balancer
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    # Azure:
    # service.beta.kubernetes.io/azure-load-balancer-resource-group: "myresourcegroup"
    # GCP:
    # cloud.google.com/load-balancer-type: "External"
```

## Step 2: Monitor LoadBalancer Provisioning

```bash
# Watch for the external IP to be assigned
kubectl get svc portainer -n portainer -w

# Initially shows:
# NAME        TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
# portainer   LoadBalancer   10.96.1.100   <pending>     9443:30779/TCP   30s

# After cloud provisioner assigns IP:
# NAME        TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)          AGE
# portainer   LoadBalancer   10.96.1.100   203.0.113.50   9443:30779/TCP   90s
```

This may take 1-5 minutes depending on the cloud provider.

## Step 3: Access Portainer

```bash
# Get the external IP
EXTERNAL_IP=$(kubectl get svc portainer -n portainer \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Access Portainer at: https://$EXTERNAL_IP:9443"
```

## Step 4: Configure AWS NLB with SSL Termination

For AWS EKS with HTTPS:

```yaml
service:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-east-1:123456:certificate/abc123"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
```

## Step 5: Configure Azure AKS LoadBalancer

```yaml
service:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: "/api/system/status"
    service.beta.kubernetes.io/azure-dns-label-name: "myportainer"  # portainer.eastus.cloudapp.azure.com
```

## Step 6: Configure GKE LoadBalancer

```yaml
service:
  type: LoadBalancer
  # For internal load balancer (VPC only):
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
    # Remove for external:
```

## Step 7: Set Up MetalLB for On-Premises (Optional)

For on-premises Kubernetes without a cloud provider:

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

# Wait for MetalLB pods
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

```yaml
# metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: portainer-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.200-192.168.1.210    # IP range for LoadBalancers

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: portainer-l2
  namespace: metallb-system
```

```bash
kubectl apply -f metallb-config.yaml

# Now LoadBalancer services get IPs from the pool
kubectl get svc portainer -n portainer
# EXTERNAL-IP: 192.168.1.200
```

## Step 8: Configure DNS for the LoadBalancer IP

After getting the external IP:

```bash
# Add a DNS A record pointing to the LoadBalancer IP
# portainer.example.com → 203.0.113.50

# Verify
nslookup portainer.example.com
# Server: 8.8.8.8
# Address: 203.0.113.50
```

Access Portainer at: `https://portainer.example.com:9443`

## Step 9: Secure the LoadBalancer

Restrict LoadBalancer access to specific IP ranges:

```yaml
service:
  type: LoadBalancer
  loadBalancerSourceRanges:
    - "203.0.113.0/24"     # Your office IP range
    - "198.51.100.0/24"    # VPN subnet
```

Or use a NetworkPolicy to restrict pod access:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: portainer-network-policy
  namespace: portainer
spec:
  podSelector:
    matchLabels:
      app: portainer
  ingress:
    - from:
        - ipBlock:
            cidr: 203.0.113.0/24    # Allowed source IPs
      ports:
        - protocol: TCP
          port: 9443
```

## Troubleshooting

### LoadBalancer Stuck in Pending

```bash
# Check events
kubectl describe svc portainer -n portainer

# Common causes:
# - Cloud provider quota exceeded
# - Missing cloud controller manager
# - MetalLB not configured (on-prem)
```

### External IP Not Accessible

```bash
# Verify from outside the cluster
curl -k https://203.0.113.50:9443/api/system/status

# Check security groups/firewall rules allow ports 9443, 9000
```

## Conclusion

LoadBalancer service type is the production-standard way to expose Portainer on cloud-hosted Kubernetes clusters. Cloud providers automatically provision and manage the load balancer, providing a stable external IP with health checking. For on-premises clusters, MetalLB provides equivalent functionality. Always configure source IP restrictions and DNS to make your Portainer deployment both accessible and secure.
