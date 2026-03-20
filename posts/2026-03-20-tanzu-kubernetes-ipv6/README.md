# How to Configure VMware Tanzu Kubernetes with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Tanzu, VMware, IPv6, Kubernetes, Antrea, Dual-Stack

Description: A guide to configuring VMware Tanzu Kubernetes Grid (TKG) with IPv6 and dual-stack networking using Antrea CNI and NSX-T Advanced Load Balancer.

VMware Tanzu Kubernetes Grid supports dual-stack IPv4/IPv6 networking using Antrea as the CNI plugin. This guide covers TKG cluster configuration for IPv6, NSX-T integration, and workload verification.

## Tanzu Kubernetes Grid IPv6 Requirements

- TKG 2.1+ for dual-stack support
- Antrea CNI (included with TKG)
- vSphere 7.0 U3+ or vSphere 8 with NSX-T
- Nodes must have dual-stack IP addresses

## TKG Management Cluster with Dual-Stack

```yaml
# ~/.config/tanzu/tkg/clusterconfigs/management-cluster.yaml

CLUSTER_NAME: mgmt-cluster
CLUSTER_PLAN: prod

# Network configuration

SERVICE_CIDR: "100.64.0.0/13,fd00:svc::/108"
CLUSTER_CIDR: "100.96.0.0/11,fd00:pod::/48"

# Antrea CNI (required for dual-stack)
CNI: antrea

# Node network
NETWORK: "VM Network"

# Enable IPv6
ENABLE_IPV6: true
```

```bash
# Initialize management cluster
tanzu management-cluster create \
  --file management-cluster.yaml \
  --timeout 60m

# Verify management cluster
tanzu management-cluster get
kubectl get nodes -o wide
```

## Workload Cluster with IPv6

```yaml
# workload-cluster.yaml

apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: workload-ipv6
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
        - "100.96.0.0/11"
        - "fd00:pod::/48"
    services:
      cidrBlocks:
        - "100.64.0.0/13"
        - "fd00:svc::/108"
  topology:
    class: tkg-vsphere-default-v1.1.0
    version: v1.27.5+vmware.1
    variables:
      - name: network
        value:
          ipFamily: ipv4,ipv6
```

```bash
# Create workload cluster
tanzu cluster create workload-ipv6 --file workload-cluster.yaml

# Get kubeconfig for workload cluster
tanzu cluster kubeconfig get workload-ipv6 --admin --export-file workload-kubeconfig.yaml

# Verify dual-stack
KUBECONFIG=workload-kubeconfig.yaml kubectl get nodes -o wide
```

## Antrea Dual-Stack Configuration

Antrea is configured by the antrea-config ConfigMap:

```yaml
# Check current Antrea configuration
kubectl get configmap antrea-config -n kube-system -o yaml

# Key settings for dual-stack:
# antrea-agent.conf:
#   featureGates:
#     AntreaIPAM: true
# antrea-controller.conf:
#   featureGates:
#     AntreaPolicy: true
```

```bash
# Verify Antrea pods are running
kubectl get pods -n kube-system -l app=antrea

# Check Antrea agent logs for IPv6 activity
kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system -l component=antrea-agent -o name | head -1) \
  | grep -i "ipv6\|dual" | tail -20

# Check antctl (Antrea CLI) for network info
kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=antrea-agent -o name | head -1) \
  -- antctl get networkpolicy
```

## NSX-T Integration with IPv6

When using TKG with NSX-T, IPv6 is configured in NSX-T Manager:

```bash
# Verify NSX-T node network adapter has IPv6
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{.status.addresses}{"\n\n"}{end}'

# NSX-T segments should be configured for dual-stack
# This is done in NSX-T Manager UI:
# Networking > Segments > <segment> > Subnets > Add IPv6 subnet

# Verify overlay tunnel is working (NSX-T uses GENEVE encapsulation)
kubectl exec -n kube-system \
  $(kubectl get pod -n kube-system -l component=antrea-agent -o name | head -1) \
  -- antctl get ovsflows | grep -i ipv6
```

## Dual-Stack Service Deployment

```yaml
# Service with dual-stack in Tanzu cluster
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  ipFamilyPolicy: RequireDualStack
  ipFamilies:
    - IPv4
    - IPv6
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
# LoadBalancer service (requires NSX-T ALB or AKO)
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
```

```bash
kubectl apply -f services.yaml

# Check dual-stack ClusterIP
kubectl get svc my-app-svc -o jsonpath='{.spec.clusterIPs}'
# Output: ["100.x.x.x","fd00:svc::x"]
```

## Verifying IPv6 in Tanzu Workloads

```bash
# Deploy a diagnostic pod
kubectl run netshoot --image=nicolaka/netshoot --restart=Never -- sleep 3600

# Check IPv6 address
kubectl exec netshoot -- ip -6 addr show
kubectl exec netshoot -- ip -6 route show

# Test IPv6 connectivity between pods
kubectl exec netshoot -- ping6 -c 3 fd00:pod::another-pod

# Test DNS resolves AAAA records
kubectl exec netshoot -- nslookup -type=AAAA kubernetes.default.svc.cluster.local

# Test IPv6 service connectivity
kubectl exec netshoot -- curl -6 http://[fd00:svc::x]/

# Cleanup
kubectl delete pod netshoot
```

## Troubleshooting Tanzu IPv6

```bash
# Check TKG cluster health
tanzu cluster list --include-management-cluster

# Check Antrea for IPv6-specific issues
kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system -l component=antrea-controller -o name) \
  | grep -i "error\|ipv6" | tail -20

# Verify vSphere VM has IPv6 address (check VM hardware/VMware Tools)
# The VM must receive an IPv6 address from the network layer

# Check kube-apiserver logs for dual-stack issues
kubectl logs -n kube-system kube-apiserver-<control-plane-node> \
  | grep -i "ipv6\|dual-stack" | tail -20
```

Tanzu Kubernetes Grid's dual-stack support with Antrea CNI provides robust IPv6 networking for enterprise vSphere environments. The key is configuring both pod and service CIDRs with IPv6 ranges at cluster creation time, as Tanzu follows Kubernetes upstream dual-stack semantics.
