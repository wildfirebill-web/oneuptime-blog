# How to Configure Rancher with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, IPv6, Kubernetes, Dual-Stack, RKE2, Networking

Description: A guide to configuring Rancher and downstream Kubernetes clusters for IPv6 and dual-stack networking, covering RKE2 cluster provisioning, network plugin selection, and Rancher server IPv6 access.

Rancher supports IPv6 through its managed Kubernetes distributions (RKE2, K3s) and via custom cluster provisioning. Dual-stack is the recommended approach, providing both IPv4 and IPv6 connectivity.

## Rancher Server with IPv6 Access

```bash
# Run Rancher server listening on IPv6 (and IPv4)

docker run -d \
  --restart=unless-stopped \
  -p 80:80 \
  -p 443:443 \
  --privileged \
  rancher/rancher:latest

# The container runtime handles dual-stack binding
# Access via IPv6: https://[2001:db8::rancher-host]
```

For production, deploy Rancher on an RKE2 or K3s cluster and configure the ingress controller with IPv6:

```yaml
# Ingress for Rancher with IPv6
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rancher
  namespace: cattle-system
spec:
  rules:
    - host: rancher.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rancher
                port:
                  number: 443
```

## Provisioning an RKE2 Dual-Stack Cluster via Rancher

In the Rancher UI, navigate to Cluster Management > Create > RKE2/K3s:

```yaml
# RKE2 cluster config (via Rancher UI or API)
# Under "Cluster Configuration" > "Networking"
kind: RKE2Config
spec:
  kubernetesVersion: v1.29.0+rke2r1
  cni: calico    # Calico supports dual-stack
  # Set cluster-cidr and service-cidr in kube-apiserver args:
  machineGlobalConfig:
    cluster-cidr: "10.42.0.0/16,fd00:pod::/48"
    service-cidr: "10.43.0.0/16,fd00:svc::/108"
```

Using the Rancher UI:
1. Cluster Management > Create > Custom
2. Select RKE2 and Calico as CNI
3. Under "Advanced" > "Additional API Server Args": add `--service-cluster-ip-range=10.43.0.0/16,fd00:svc::/108`

## RKE2 Cluster Config File (Manual)

```yaml
# /etc/rancher/rke2/config.yaml on the first server node

# Dual-stack pod and service CIDRs
cluster-cidr:
  - 10.42.0.0/16
  - fd00:pod::/48

service-cidr:
  - 10.43.0.0/16
  - fd00:svc::/108

# CNI that supports dual-stack
cni: calico

# Bind to all interfaces (IPv4 and IPv6)
bind-address: "::"
```

```bash
# Start RKE2 server
systemctl enable --now rke2-server

# Check cluster dual-stack
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
kubectl get nodes -o wide

# Verify pod CIDR includes IPv6
kubectl get node <node-name> -o jsonpath='{.spec.podCIDRs}'
```

## K3s Dual-Stack Cluster via Rancher

```bash
# K3s server with dual-stack (for imported cluster or standalone)
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-cidr=10.42.0.0/16,fd00:pod::/48 \
  --service-cidr=10.43.0.0/16,fd00:svc::/108 \
  --flannel-ipv6-masq \
  --disable=traefik   # Optional: use own ingress

# Import the K3s cluster into Rancher
# Rancher UI: Cluster Management > Import Existing > Generic
# Apply the generated kubectl apply command on the K3s cluster
```

## Calico IPv6 Configuration in Rancher-Managed Cluster

```bash
# After cluster creation, configure Calico for dual-stack
kubectl apply -f - <<'EOF'
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: ipv6-ippool
spec:
  cidr: fd00:pod::/48
  ipipMode: Never
  vxlanMode: Never
  natOutgoing: true
  nodeSelector: all()
EOF

# Verify Calico has both IP pools
kubectl get ippools
```

## Verifying IPv6 in Rancher-Managed Cluster

```bash
# Check nodes have dual-stack CIDRs
kubectl get nodes -o custom-columns=NAME:.metadata.name,CIDRS:.spec.podCIDRs

# Deploy a test pod and check IPv6
kubectl run test-ipv6 --image=nicolaka/netshoot --restart=Never -- sleep 3600
kubectl exec test-ipv6 -- ip -6 addr show
kubectl exec test-ipv6 -- ping6 -c 3 fd00:svc::a   # Ping a service IPv6

# Check that services get dual-stack ClusterIPs
kubectl create service clusterip my-svc --tcp=80:80
kubectl get svc my-svc -o wide

# Cleanup
kubectl delete pod test-ipv6
kubectl delete svc my-svc
```

## Rancher Monitoring with IPv6

```bash
# Install Rancher Monitoring chart (Prometheus + Grafana)
# It works with dual-stack clusters automatically

# Verify Prometheus scrapes over IPv6
kubectl get svc -n cattle-monitoring-system rancher-monitoring-prometheus \
  -o jsonpath='{.spec.clusterIPs}'

# Access Grafana (port-forward works with dual-stack)
kubectl port-forward -n cattle-monitoring-system svc/rancher-monitoring-grafana 3000:80
```

Rancher's support for IPv6 is primarily delivered through RKE2 and K3s with Calico or Flannel CNI plugins. Configure dual-stack CIDRs at cluster creation time, as changing them after the fact requires cluster recreation.
