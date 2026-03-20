# How to Deploy IPv6 Applications on Google Distributed Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Google Distributed Cloud, Kubernetes, IPv6, GDC, Networking, Bare Metal

Description: A guide to deploying IPv6-enabled applications on Google Distributed Cloud (GDC) Bare Metal, including cluster configuration and load balancer setup.

Google Distributed Cloud (GDC) Bare Metal extends Google Kubernetes Engine to on-premises infrastructure. It supports IPv6 and dual-stack networking, enabling organizations to deploy IPv6 workloads on their own hardware while retaining GKE-compatible APIs.

## Prerequisites

- A Google Distributed Cloud Bare Metal installation
- Admin workstation with `bmctl` CLI installed
- Target nodes with IPv6 addresses assigned on their network interfaces

## Step 1: Configure the Cluster for Dual-Stack

When creating or updating a GDC Bare Metal cluster, set dual-stack in the cluster configuration file:

```yaml
# cluster-config.yaml - GDC Bare Metal dual-stack cluster configuration

apiVersion: baremetal.cluster.gke.io/v1
kind: Cluster
metadata:
  name: my-ipv6-cluster
  namespace: cluster-my-ipv6-cluster
spec:
  type: standalone
  # Enable dual-stack address families
  clusterNetwork:
    pods:
      cidrBlocks:
        - 192.168.0.0/16
        - fd00:1234::/80      # IPv6 pod CIDR
    services:
      cidrBlocks:
        - 10.96.0.0/20
        - fd00:1234:1::/112   # IPv6 service CIDR
  # Load balancer must support IPv6 virtual IPs
  loadBalancer:
    mode: bundled
    ports:
      controlPlaneLBPort: 443
    vips:
      controlPlaneVIP: "10.0.0.8"
      ingressVIP: "10.0.0.9"
```

## Step 2: Create the Cluster

```bash
# Create the cluster using the bmctl CLI
bmctl create cluster -c my-ipv6-cluster --config cluster-config.yaml

# Monitor cluster creation progress
bmctl check cluster -c my-ipv6-cluster
```

## Step 3: Deploy an IPv6 Application

Once the cluster is running, deploy a workload and expose it via a dual-stack Service:

```yaml
# ipv6-app.yaml - Deployment and dual-stack Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ipv6-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ipv6-nginx
  template:
    metadata:
      labels:
        app: ipv6-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: ipv6-nginx-svc
spec:
  selector:
    app: ipv6-nginx
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv4
    - IPv6
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
```

```bash
kubectl apply -f ipv6-app.yaml
```

## Step 4: Verify Load Balancer Assignment

```bash
# Check that the service has both IPv4 and IPv6 LoadBalancer IPs
kubectl get svc ipv6-nginx-svc -o wide

# Get the ClusterIPs to verify dual-stack assignment
kubectl get svc ipv6-nginx-svc -o jsonpath='{.spec.clusterIPs}'
```

## Step 5: Test IPv6 Connectivity

```bash
# Get the external IPv6 load balancer IP
LB_IPV6=$(kubectl get svc ipv6-nginx-svc \
  -o jsonpath='{.status.loadBalancer.ingress[?(@.ipFamily=="IPv6")].ip}')

# Test HTTP access via IPv6
curl -6 http://[$LB_IPV6]/
```

## Step 6: Configure Network Policies for IPv6

GDC Bare Metal with Calico supports IPv6 network policies:

```yaml
# allow-ipv6-ingress.yaml - Allow IPv6 HTTP traffic to the application
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ipv6-http
spec:
  podSelector:
    matchLabels:
      app: ipv6-nginx
  ingress:
    - ports:
        - protocol: TCP
          port: 80
  policyTypes:
    - Ingress
```

## Monitoring

Use OneUptime to monitor the IPv6 endpoints of your GDC-deployed applications with uptime checks that verify both IPv4 and IPv6 reachability:

```bash
# Quick connectivity test from outside the cluster
ping6 $LB_IPV6
curl -6 -o /dev/null -s -w "%{http_code}" http://[$LB_IPV6]/
```

GDC Bare Metal's dual-stack support allows organizations running their own on-premises hardware to fully embrace IPv6 while using familiar Kubernetes APIs.
