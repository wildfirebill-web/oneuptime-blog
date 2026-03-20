# How to Configure IPv6 LoadBalancer Services in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv6, LoadBalancer, Service, Dual-Stack, Cloud Load Balancer

Description: Configure Kubernetes LoadBalancer Services to receive IPv6 external IPs, set up dual-stack load balancers on AWS, GCP, and Azure, and verify external IPv6 connectivity to Kubernetes services.

## Introduction

Kubernetes LoadBalancer Services provision external load balancers through the cloud controller manager. In dual-stack clusters, LoadBalancer Services can receive both IPv4 and IPv6 external IPs. Cloud-specific annotations control whether the provisioned load balancer supports IPv6. The `status.loadBalancer.ingress` field shows all assigned external IPs including IPv6 addresses.

## Create Dual-Stack LoadBalancer Service

```yaml
# lb-dual-stack.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  annotations:
    # AWS: dual-stack NLB
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-ip-address-type: "dualstack"
    # GCP: (no annotation needed, uses ipFamilyPolicy)
    # Azure: (no annotation needed, uses Standard SKU)
spec:
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
  ipFamilyPolicy: PreferDualStack
  ipFamilies: [IPv4, IPv6]
  type: LoadBalancer
```

```bash
kubectl apply -f lb-dual-stack.yaml

# Wait for external IP assignment
kubectl get svc web-lb -w

# Check for IPv6 external IP
kubectl get svc web-lb -o jsonpath='{.status.loadBalancer.ingress}'
# [{"ip":"34.x.x.x"},{"ip":"2600:1900:..."}]
```

## AWS EKS with IPv6 Load Balancer

```yaml
# EKS: Use AWS Load Balancer Controller for dual-stack NLB
apiVersion: v1
kind: Service
metadata:
  name: web-lb-aws
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-ip-address-type: "dualstack"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
  ipFamilyPolicy: PreferDualStack
  ipFamilies: [IPv4, IPv6]
  type: LoadBalancer
```

## GCP GKE with IPv6 Load Balancer

```yaml
# GKE: dual-stack with external IPv6
apiVersion: v1
kind: Service
metadata:
  name: web-lb-gcp
  annotations:
    # Standard external LB with IPv6
    cloud.google.com/l4-rbs: "enabled"
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
  ipFamilyPolicy: PreferDualStack
  ipFamilies: [IPv4, IPv6]
  type: LoadBalancer
```

## Azure AKS with IPv6 Load Balancer

```yaml
# AKS: dual-stack LoadBalancer (requires dual-stack cluster)
apiVersion: v1
kind: Service
metadata:
  name: web-lb-azure
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
  ipFamilyPolicy: PreferDualStack
  ipFamilies: [IPv4, IPv6]
  type: LoadBalancer
```

## Test External IPv6 Connectivity

```bash
# Get LoadBalancer external IPs
kubectl get svc web-lb -o jsonpath='{.status.loadBalancer.ingress[*].ip}'

# Find IPv6 external IP
LB_IPV6=$(kubectl get svc web-lb \
    -o jsonpath='{range .status.loadBalancer.ingress[*]}{.ip}{"\n"}{end}' | \
    grep ":")

echo "LoadBalancer IPv6: $LB_IPV6"

# Test HTTP over IPv6
curl -6 "http://[$LB_IPV6]/"

# Test HTTPS over IPv6
curl -6 "https://[$LB_IPV6]/" --insecure

# Add DNS records for your domain
# A record -> IPv4 LB IP
# AAAA record -> IPv6 LB IP

# Test via DNS
curl -6 https://example.com/
```

## Troubleshoot LoadBalancer IPv6 Assignment

```bash
# Check events for LB provisioning errors
kubectl describe svc web-lb | grep -A20 Events

# Check cloud controller manager logs
kubectl -n kube-system logs daemonset/cloud-controller-manager 2>/dev/null | \
    grep -i "loadbalancer\|ipv6"

# For EKS, check AWS LB Controller
kubectl -n kube-system logs deployment/aws-load-balancer-controller | \
    grep -i "dualstack\|ipv6"

# Verify service spec
kubectl get svc web-lb -o yaml | grep -A10 "ipFamily"
```

## Conclusion

Kubernetes LoadBalancer Services receive external IPv6 IPs when the cloud provider supports dual-stack load balancers and the service uses `ipFamilyPolicy: PreferDualStack`. On AWS, use the AWS Load Balancer Controller with `aws-load-balancer-ip-address-type: dualstack` annotation. On GCP GKE, dual-stack is controlled by `ipFamilyPolicy`. On Azure AKS, dual-stack clusters automatically provision dual-stack Standard Load Balancers. Check `status.loadBalancer.ingress` for external IPv6 IPs and add AAAA DNS records pointing to the IPv6 load balancer IP for full IPv6 accessibility.
