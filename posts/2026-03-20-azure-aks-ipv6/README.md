# How to Configure IPv6 on Azure Kubernetes Service (AKS)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, IPv6, AKS, Kubernetes, Dual-Stack, Container Networking

Description: Configure Azure AKS clusters with dual-stack IPv4/IPv6 networking, enabling pods and services to have both IPv4 and IPv6 addresses for native dual-stack Kubernetes deployments.

## Introduction

Azure AKS supports dual-stack IPv4/IPv6 Kubernetes networking. In dual-stack mode, pods receive both IPv4 and IPv6 addresses, services can have both ClusterIP families, and load balancers can front traffic from IPv4 and IPv6 clients. This is built on Azure CNI with dual-stack VNet subnet assignment.

## Create Dual-Stack AKS Cluster

```bash
RG="rg-aks-ipv6"
LOCATION="eastus"
CLUSTER_NAME="aks-dualstack"

# Create resource group

az group create --name "$RG" --location "$LOCATION"

# Create dual-stack VNet
az network vnet create \
    --resource-group "$RG" \
    --name vnet-aks \
    --address-prefixes "10.0.0.0/8" "fd00:aks::/48"

az network vnet subnet create \
    --resource-group "$RG" \
    --vnet-name vnet-aks \
    --name subnet-aks \
    --address-prefixes "10.1.0.0/16" "fd00:aks:1::/64"

SUBNET_ID=$(az network vnet subnet show \
    --resource-group "$RG" \
    --vnet-name vnet-aks \
    --name subnet-aks \
    --query id --output tsv)

# Create dual-stack AKS cluster
az aks create \
    --resource-group "$RG" \
    --name "$CLUSTER_NAME" \
    --location "$LOCATION" \
    --node-count 3 \
    --node-vm-size Standard_D2s_v3 \
    --network-plugin azure \
    --network-plugin-mode overlay \
    --ip-families IPv4,IPv6 \
    --pod-cidr "192.168.0.0/16" "fd12:3456:789a::/48" \
    --service-cidrs "10.0.0.0/16" "fd12:3456:789b::/108" \
    --dns-service-ip "10.0.0.10" \
    --vnet-subnet-id "$SUBNET_ID" \
    --generate-ssh-keys
```

## Configure kubectl for AKS IPv6

```bash
# Get kubeconfig
az aks get-credentials \
    --resource-group "$RG" \
    --name "$CLUSTER_NAME"

# Check nodes have dual-stack addresses
kubectl get nodes -o wide
# INTERNAL-IP should show both IPv4 and IPv6

# Check pod CIDR assignments
kubectl describe node | grep -A 5 "Addresses:"

# Check system pods have dual-stack IPs
kubectl get pods -n kube-system -o wide
```

## Deploy Dual-Stack Service

```yaml
# dual-stack-service.yaml
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
  # Dual-stack service (both IPv4 and IPv6 ClusterIPs)
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv4
    - IPv6
  type: ClusterIP

---
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  namespace: default
  annotations:
    # Azure Load Balancer with dualstack
    service.beta.kubernetes.io/azure-load-balancer-ipv4-frontend-ip-configuration-name: frontend-ipv4
    service.beta.kubernetes.io/azure-load-balancer-ipv6-frontend-ip-configuration-name: frontend-ipv6
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  ipFamilyPolicy: PreferDualStack
  ipFamilies:
    - IPv4
    - IPv6
  type: LoadBalancer
```

```bash
# Apply and verify
kubectl apply -f dual-stack-service.yaml

# Check service has both IPv4 and IPv6 ClusterIPs
kubectl get svc web-service -o wide
# CLUSTER-IP should show both families

# Check endpoints
kubectl get endpoints web-service -o yaml | grep ip
```

## Test IPv6 Pod Connectivity

```bash
# Deploy test pod
kubectl run test --image=curlimages/curl --rm -it \
    --restart=Never -- sh

# Inside pod: check addresses
ip -6 addr show

# Test IPv6 connectivity
curl -6 https://ipv6.google.com
ping6 -c 3 2001:4860:4860::8888

# Test inter-pod IPv6
# Get IPv6 of another pod
OTHER_POD_IPV6=$(kubectl get pod other-pod -o jsonpath='{.status.podIPs[?(@.ip=~".*:.*")].ip}')
curl "http://[${OTHER_POD_IPV6}]/"
```

## Conclusion

AKS dual-stack requires `--ip-families IPv4,IPv6` at cluster creation with both IPv4 and IPv6 pod CIDRs and service CIDRs. The dual-stack VNet subnet must have both IPv4 and IPv6 CIDR blocks. Services use `ipFamilyPolicy: PreferDualStack` with `ipFamilies: [IPv4, IPv6]` to get both ClusterIPs. LoadBalancer services create Azure Load Balancers with both IPv4 and IPv6 frontend configurations, enabling external clients to connect over either protocol. Once created, the IP family configuration cannot be changed - design for dual-stack from the start.
