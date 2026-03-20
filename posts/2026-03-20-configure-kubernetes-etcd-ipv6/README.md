# How to Configure Kubernetes etcd Cluster with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, etcd, IPv6, Distributed Systems, HA, Cluster Configuration

Description: Configure a highly available etcd cluster with IPv6 endpoints for Kubernetes, covering peer communication, client access, TLS, and cluster bootstrapping over IPv6.

---

etcd is the key-value store backing Kubernetes cluster state. In IPv6-enabled Kubernetes clusters, etcd must be configured to listen on IPv6 interfaces and communicate with peers over IPv6 addresses.

## etcd IPv6 Configuration Parameters

The key etcd flags for IPv6:

```bash
# etcd listen and advertise addresses for IPv6

--listen-peer-urls=https://[2001:db8::1]:2380
--listen-client-urls=https://[2001:db8::1]:2379,https://[::1]:2379
--initial-advertise-peer-urls=https://[2001:db8::1]:2380
--advertise-client-urls=https://[2001:db8::1]:2379
```

## Setting Up a 3-Node etcd Cluster with IPv6

### Node 1 (2001:db8::1)

```bash
etcd \
  --name etcd1 \
  --data-dir /var/lib/etcd \
  \
  --listen-peer-urls "https://[2001:db8::1]:2380" \
  --initial-advertise-peer-urls "https://[2001:db8::1]:2380" \
  \
  --listen-client-urls "https://[2001:db8::1]:2379,https://[::1]:2379" \
  --advertise-client-urls "https://[2001:db8::1]:2379" \
  \
  --initial-cluster-token "etcd-cluster-1" \
  --initial-cluster \
    "etcd1=https://[2001:db8::1]:2380,etcd2=https://[2001:db8::2]:2380,etcd3=https://[2001:db8::3]:2380" \
  --initial-cluster-state new \
  \
  --cert-file=/etc/etcd/certs/server.crt \
  --key-file=/etc/etcd/certs/server.key \
  --peer-cert-file=/etc/etcd/certs/peer.crt \
  --peer-key-file=/etc/etcd/certs/peer.key \
  --trusted-ca-file=/etc/etcd/certs/ca.crt \
  --peer-trusted-ca-file=/etc/etcd/certs/ca.crt \
  --peer-client-cert-auth \
  --client-cert-auth
```

## Generating TLS Certificates with IPv6 SANs

```bash
# Create etcd CA
cfssl gencert -initca ca-config.json | cfssljson -bare ca

# Create server cert config with IPv6 SANs
cat > server-csr.json << 'EOF'
{
  "CN": "etcd",
  "key": {"algo": "rsa", "size": 2048},
  "hosts": [
    "2001:db8::1",
    "2001:db8::2",
    "2001:db8::3",
    "localhost",
    "127.0.0.1",
    "::1"
  ]
}
EOF

# Generate server certificate
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=server \
  server-csr.json | cfssljson -bare server
```

## etcd Systemd Service for IPv6

```ini
# /etc/systemd/system/etcd.service
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network-online.target

[Service]
Type=notify
EnvironmentFile=/etc/etcd/etcd.env
ExecStart=/usr/local/bin/etcd \
  --name=${ETCD_NAME} \
  --data-dir=/var/lib/etcd \
  --listen-peer-urls=https://[${ETCD_IP}]:2380 \
  --initial-advertise-peer-urls=https://[${ETCD_IP}]:2380 \
  --listen-client-urls=https://[${ETCD_IP}]:2379,https://[::1]:2379 \
  --advertise-client-urls=https://[${ETCD_IP}]:2379 \
  --initial-cluster-token=etcd-cluster \
  --initial-cluster=${ETCD_INITIAL_CLUSTER} \
  --initial-cluster-state=new \
  --cert-file=/etc/etcd/certs/server.crt \
  --key-file=/etc/etcd/certs/server.key \
  --peer-cert-file=/etc/etcd/certs/peer.crt \
  --peer-key-file=/etc/etcd/certs/peer.key \
  --trusted-ca-file=/etc/etcd/certs/ca.crt \
  --peer-trusted-ca-file=/etc/etcd/certs/ca.crt \
  --client-cert-auth \
  --peer-client-cert-auth
Restart=on-failure
User=etcd

[Install]
WantedBy=multi-user.target
```

## Configuring Kubernetes to Use IPv6 etcd

In kubeadm config, specify IPv6 etcd endpoints:

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
etcd:
  external:
    endpoints:
      - "https://[2001:db8::1]:2379"
      - "https://[2001:db8::2]:2379"
      - "https://[2001:db8::3]:2379"
    caFile: "/etc/etcd/certs/ca.crt"
    certFile: "/etc/kubernetes/pki/apiserver-etcd-client.crt"
    keyFile: "/etc/kubernetes/pki/apiserver-etcd-client.key"
networking:
  podSubnet: "fd00:10::/64"
  serviceSubnet: "fd00:20::/112"
```

## Verifying etcd Cluster Health over IPv6

```bash
# Set etcdctl environment
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS="https://[2001:db8::1]:2379"
export ETCDCTL_CACERT="/etc/etcd/certs/ca.crt"
export ETCDCTL_CERT="/etc/etcd/certs/client.crt"
export ETCDCTL_KEY="/etc/etcd/certs/client.key"

# Check cluster health
etcdctl endpoint health --cluster

# Check cluster members
etcdctl member list

# Test write and read
etcdctl put /test/ipv6 "hello from IPv6"
etcdctl get /test/ipv6
```

Configuring etcd with IPv6 addresses provides the foundation for IPv6-native Kubernetes clusters, ensuring all control plane communications including state storage and leader election operate natively over IPv6.
