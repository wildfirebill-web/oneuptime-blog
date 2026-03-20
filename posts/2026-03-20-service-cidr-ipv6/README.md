# How to Configure Service CIDR for IPv6 in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv6, Service CIDR, ClusterIP, Dual-Stack, Kube-apiserver

Description: Configure IPv6 Service CIDR ranges in Kubernetes clusters, understand how ClusterIPs are allocated from service CIDRs, and verify that Services receive IPv6 ClusterIPs from the configured ranges.

## Introduction

The service CIDR in Kubernetes defines the IP range for ClusterIPs assigned to Services. In dual-stack clusters, the service CIDR includes both IPv4 and IPv6 ranges. The kube-apiserver allocates ClusterIPs from these ranges when Services are created. The IPv6 service CIDR must be sized appropriately for the number of services in the cluster - a `/108` provides 1,048,576 addresses, sufficient for most deployments.

## Configure Service CIDR in kubeadm

```yaml
# kubeadm-config.yaml - service CIDR sizing

apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.0
networking:
  podSubnet: "10.244.0.0/16,fd00:10:244::/56"
  # IPv4 /12 = 1M service IPs
  # IPv6 /108 = 1M service IPs (2^20 = 1,048,576)
  serviceSubnet: "10.96.0.0/12,fd00:10:96::/108"
apiServer:
  extraArgs:
    service-cluster-ip-range: "10.96.0.0/12,fd00:10:96::/108"
controllerManager:
  extraArgs:
    cluster-cidr: "10.244.0.0/16,fd00:10:244::/56"
    service-cluster-ip-range: "10.96.0.0/12,fd00:10:96::/108"
    node-cidr-mask-size-ipv4: "24"
    node-cidr-mask-size-ipv6: "64"
```

## View Current Service CIDR

```bash
# Check the configured service CIDR from kube-apiserver
kubectl -n kube-system get pod kube-apiserver-<node> -o yaml | \
    grep "service-cluster-ip-range"

# Or check from kube-controller-manager
kubectl -n kube-system get pod kube-controller-manager-<node> -o yaml | \
    grep "service-cluster-ip-range"

# View how many services exist vs capacity
kubectl get svc -A --no-headers | wc -l
# If this approaches your service CIDR size, expand soon

# Check kubernetes service (first IP in service CIDR)
kubectl get svc kubernetes -o jsonpath='{.spec.clusterIPs}'
# ["10.96.0.1","fd00:10:96::1"]
```

## IPv6 Service CIDR Sizing Guide

```text
Service CIDR Size Reference:
  IPv6 /108:  2^20 = 1,048,576 service IPs     (recommended for production)
  IPv6 /112:  2^16 = 65,536 service IPs         (small/medium clusters)
  IPv6 /116:  2^12 = 4,096 service IPs          (development/testing)

Minimum IPv6 service CIDR prefix: /108 for >65K services
The IPv6 service CIDR must not overlap with pod CIDRs or node CIDRs.

Choose from ULA range:
  fd00:10:96::/108  (aligned with 10.96.0.0/12 IPv4 range)
  fd00:svc::/108    (semantic naming)
```

## Create Services and Verify IPv6 ClusterIP Allocation

```bash
# Create multiple services and verify IPv6 allocation
for i in $(seq 1 5); do
    kubectl create service clusterip "svc-$i" \
        --tcp=80:80 \
        --dry-run=client -o yaml | \
        kubectl patch --dry-run=client -f - \
        -p '{"spec":{"ipFamilyPolicy":"PreferDualStack"}}' | \
        kubectl apply -f -
done

# View allocated ClusterIPs
kubectl get svc -l "": -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.clusterIPs}{"\n"}{end}'

# Clean up
for i in $(seq 1 5); do kubectl delete svc "svc-$i"; done
```

## Monitor Service IP Exhaustion

```bash
# Count allocated service IPs
TOTAL_SVCS=$(kubectl get svc -A --no-headers | wc -l)
echo "Total services: $TOTAL_SVCS"

# IPv6 /108 capacity
CAPACITY=$((2**20))
echo "Service CIDR capacity: $CAPACITY"

USAGE_PCT=$((TOTAL_SVCS * 100 / CAPACITY))
echo "Usage: ${USAGE_PCT}%"

# Alert if >80% full
if [ "$USAGE_PCT" -gt 80 ]; then
    echo "WARNING: Service CIDR is ${USAGE_PCT}% full!"
fi
```

## Conclusion

Configure the IPv6 service CIDR in `kubeadm-config.yaml` under `serviceSubnet` with a comma-separated IPv4 and IPv6 CIDR. Pass the same value to kube-apiserver's `service-cluster-ip-range` and kube-controller-manager's matching flag. Use a `/108` IPv6 service CIDR for production clusters (1M addresses). The first IP in the service CIDR is reserved for the `kubernetes` service. Service CIDRs cannot be changed after cluster initialization without migrating all services, so size them generously at creation time.
