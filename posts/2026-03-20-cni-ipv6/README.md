# How to Configure CNI Plugins for IPv6 (Calico, Cilium, Flannel)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv6, CNI, Calico, Cilium, Flannel, Networking

Description: Configure popular Kubernetes CNI plugins - Calico, Cilium, and Flannel - for IPv6 and dual-stack networking, with complete installation and configuration examples for each plugin.

## Introduction

Kubernetes requires a CNI (Container Network Interface) plugin for pod networking. IPv6 support varies across CNI plugins - Calico and Cilium have mature dual-stack support, while Flannel has basic dual-stack support. Each plugin requires separate configuration for the IPv6 address pools. This guide covers configuration for the three most popular CNI plugins in dual-stack Kubernetes clusters.

## Calico with Dual-Stack

```bash
# Install Tigera operator

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# Configure Calico with dual-stack IP pools
kubectl apply -f - << 'EOF'
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      # IPv4 pool
      - blockSize: 26
        cidr: 10.244.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
      # IPv6 pool
      - blockSize: 122
        cidr: fd00:10:244::/56
        encapsulation: VXLAN
        natOutgoing: Enabled
        nodeSelector: all()
EOF

# Verify Calico pods are running
kubectl -n calico-system get pods

# Check IP pools
kubectl get ippool -o wide
```

## Cilium with Dual-Stack

```bash
# Install Cilium CLI
curl -L --fail --remote-name-all \
    https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
sudo tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin

# Install Cilium with dual-stack
cilium install \
    --set ipv4.enabled=true \
    --set ipv6.enabled=true \
    --set tunnel=vxlan \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost=$(kubectl get node -o jsonpath='{.items[0].status.addresses[0].address}') \
    --set k8sServicePort=6443

# Verify Cilium status
cilium status

# Check dual-stack is active
cilium config view | grep -i ipv6
```

```yaml
# Cilium Helm values for dual-stack
# values.yaml
ipv4:
  enabled: true
ipv6:
  enabled: true

ipam:
  mode: kubernetes

tunnel: vxlan

# If using BGP for routing
bgp:
  enabled: true
  announce:
    loadbalancerIP: true
    podCIDR: true
```

## Flannel with Dual-Stack

```yaml
# flannel-configmap.yaml - dual-stack config
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "IPv6Network": "fd00:10:244::/56",
      "EnableIPv6": true,
      "Backend": {
        "Type": "vxlan",
        "VNI": 1,
        "Port": 4789
      }
    }
```

```bash
# Apply Flannel with dual-stack config
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
# Then patch the ConfigMap with dual-stack settings
kubectl patch configmap kube-flannel-cfg -n kube-flannel \
    --patch-file=flannel-configmap.yaml
kubectl rollout restart daemonset kube-flannel-ds -n kube-flannel
```

## Verify CNI IPv6 Configuration

```bash
# For any CNI: check pods get dual-stack addresses
kubectl run test-pod --image=alpine --command -- sleep infinity
kubectl get pod test-pod -o jsonpath='{.status.podIPs}'
# [{"ip":"10.244.x.x"},{"ip":"fd00:10:244::x"}]

# Check node routing for IPv6 (Calico/Cilium)
kubectl exec -n calico-system calico-node-xxxxx -- \
    ip -6 route show

# Cilium connectivity test
cilium connectivity test --test '//pod-to-pod'
```

## Conclusion

Configure CNI plugins for dual-stack Kubernetes by adding IPv6 IP pools alongside IPv4 pools. Calico uses `Installation` CRD with multiple `ipPools`; Cilium uses `ipv6.enabled=true` in Helm values or the CLI; Flannel uses `IPv6Network` and `EnableIPv6` in its ConfigMap. All three support VXLAN encapsulation for IPv6 tunneling. After CNI installation, verify dual-stack by checking pod IPs with `kubectl get pod -o jsonpath='{.status.podIPs}'` - you should see both IPv4 and IPv6 addresses.
