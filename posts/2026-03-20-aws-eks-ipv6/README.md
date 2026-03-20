# How to Configure IPv6 for AWS EKS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, IPv6, EKS, Kubernetes, CNI, Container Networking

Description: Configure AWS EKS clusters to use IPv6 for pod networking, enabling each pod to have a unique IPv6 address and eliminating IPv4 address exhaustion concerns.

## Introduction

AWS EKS supports IPv6 pod networking using the AWS VPC CNI plugin. In IPv6 mode, each pod gets a unique IPv6 address from the VPC's IPv6 CIDR block, eliminating the IPv4 address exhaustion problem common in large clusters. IPv6 EKS clusters are IPv6-only for pods but still use IPv4 for control plane communication and some services.

## Create IPv6 EKS Cluster

```bash
# Create EKS cluster with IPv6 IP family

eksctl create cluster \
    --name ipv6-cluster \
    --version 1.29 \
    --region us-east-1 \
    --node-type t3.medium \
    --nodes 3 \
    --ip-family IPv6

# Or using AWS CLI with eksctl config file:
cat << 'EOF' > /tmp/eks-ipv6-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ipv6-cluster
  region: us-east-1
  version: "1.29"
kubernetesNetworkConfig:
  ipFamily: IPv6
managedNodeGroups:
  - name: managed-workers
    instanceType: t3.medium
    desiredCapacity: 3
    minSize: 1
    maxSize: 5
EOF

eksctl create cluster -f /tmp/eks-ipv6-cluster.yaml
```

## Verify IPv6 Pod Networking

```bash
# Get kubeconfig
aws eks update-kubeconfig --name ipv6-cluster --region us-east-1

# Check node IPv6 addresses
kubectl get nodes -o wide
# INTERNAL-IP column should show IPv6 addresses

# Deploy a test pod
kubectl run test --image=nginx --port=80
kubectl get pod test -o wide
# POD-IP should be an IPv6 address

# Verify pod has IPv6 address
kubectl exec test -- ip -6 addr show

# Check pods in a namespace
kubectl get pods -A -o wide | grep -v "^NAMESPACE" | \
    awk '{print $2, $8}' | head -20
```

## Configure Services for IPv6

```yaml
# service-ipv6.yaml - Service with IPv6
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: default
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP

  # For IPv6-only clusters, the ClusterIP will be IPv6
  # For dual-stack clusters, specify ipFamilies
  ipFamilies:
    - IPv6
  ipFamilyPolicy: SingleStack
```

```yaml
# deployment-ipv6.yaml - Deployment targeting IPv6 service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

## Load Balancer Service for IPv6

```yaml
# lb-service-ipv6.yaml - LoadBalancer with IPv6
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  namespace: default
  annotations:
    # Use dualstack for ALB that accepts both IPv4 and IPv6
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-ip-address-type: "dualstack"
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
    - port: 443
      targetPort: 443
  type: LoadBalancer
```

## AWS Load Balancer Controller for IPv6

```bash
# Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --namespace kube-system \
    --set clusterName=ipv6-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller

# Verify installation
kubectl get deployment -n kube-system aws-load-balancer-controller
```

## Troubleshoot IPv6 Pod Networking

```bash
# Check VPC CNI plugin configuration
kubectl -n kube-system get configmap amazon-vpc-cni -o yaml

# Check CNI IP family setting
kubectl -n kube-system get configmap amazon-vpc-cni -o jsonpath='{.data.IP_FAMILY}'
# Should return "IPv6"

# Check pod IP allocation
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.podIP}{"\n"}{end}'

# Test IPv6 pod-to-pod communication
kubectl exec pod-a -- ping6 -c 3 <pod-b-ipv6-address>
```

## Conclusion

AWS EKS IPv6 mode assigns each pod a unique IPv6 address from the VPC's `/56` block, eliminating IPv4 exhaustion. Enable IPv6 at cluster creation with `--ip-family IPv6` - it cannot be changed after creation. Services in IPv6 clusters get IPv6 ClusterIPs. Use the AWS Load Balancer Controller with the `dualstack` annotation for ALBs that accept IPv6 client connections. Verify pod networking with `kubectl get pods -o wide` and confirm IPv6 addresses appear in the POD-IP column.
