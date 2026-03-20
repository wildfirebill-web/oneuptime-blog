# How to Rotate RKE2 Certificates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Certificates, TLS, Security, Kubernetes, Certificate Rotation, SUSE Rancher

Description: Learn how to rotate RKE2 cluster certificates manually and automatically, including the API server, etcd, and kubelet certificates, to maintain cluster security.

---

RKE2 certificates expire after one year by default. Certificate rotation is essential for maintaining cluster security and preventing unexpected outages caused by certificate expiry.

---

## Step 1: Check Certificate Expiry

Before rotating, check when your certificates expire:

```bash
# Check all RKE2 certificate expiry dates
for cert in /var/lib/rancher/rke2/server/tls/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in "$cert" -noout -dates 2>/dev/null
done

# Check specific certificates
openssl x509 -in /var/lib/rancher/rke2/server/tls/server-ca.crt -noout -dates
openssl x509 -in /var/lib/rancher/rke2/server/tls/client-ca.crt -noout -dates

# Check etcd certificates
for cert in /var/lib/rancher/rke2/server/tls/etcd/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in "$cert" -noout -dates 2>/dev/null
done
```

---

## Step 2: Automatic Certificate Renewal

RKE2 automatically rotates certificates that expire within 90 days when the server is restarted. This is the safest rotation method:

```bash
# Restart RKE2 server — certificates expiring within 90 days are renewed
systemctl restart rke2-server

# Verify certificates were renewed
openssl x509 -in /var/lib/rancher/rke2/server/tls/server-ca.crt -noout -dates
```

---

## Step 3: Force Certificate Rotation

To force rotation regardless of expiry:

```bash
# Stop RKE2 on all server nodes
systemctl stop rke2-server

# Rotate certificates on the primary server node
rke2 certificate rotate

# Start RKE2 on the primary server node
systemctl start rke2-server

# Wait for the primary to be ready
kubectl get nodes

# Start RKE2 on remaining server nodes
# (on each additional server node)
systemctl start rke2-server
```

---

## Step 4: Rotate Only Specific Certificates

To rotate a specific component's certificate:

```bash
# Rotate only the API server certificate
rke2 certificate rotate --service kube-apiserver

# Rotate only the etcd certificates
rke2 certificate rotate --service etcd

# Rotate only the kubelet certificates
rke2 certificate rotate --service kubelet
```

---

## Step 5: Update kubeconfig After Rotation

After certificate rotation, the kubeconfig file is updated automatically, but clients using cached kubeconfigs need to refresh:

```bash
# Copy the updated kubeconfig
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
chmod 600 ~/.kube/config

# Update the kubeconfig server address if needed
kubectl config set-cluster default \
  --server=https://<api-server-ip>:6443

# Verify connectivity
kubectl get nodes
```

---

## Step 6: Rotate Agent Node Certificates

Agent nodes also have certificates that need rotation:

```bash
# On each agent node, restart RKE2 agent
systemctl restart rke2-agent

# Verify the agent reconnected
# (run on a server node)
kubectl get nodes
```

---

## Step 7: Verify After Rotation

```bash
# Confirm new expiry dates
for cert in /var/lib/rancher/rke2/server/tls/*.crt; do
  echo "=== $(basename $cert) ==="
  openssl x509 -in "$cert" -noout -enddate 2>/dev/null
done

# Check cluster health
kubectl get nodes
kubectl get pods -n kube-system

# Check etcd health
/var/lib/rancher/rke2/bin/etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/rke2/server/tls/etcd/client.crt \
  --key=/var/lib/rancher/rke2/server/tls/etcd/client.key \
  endpoint health
```

---

## Scheduling Certificate Rotation

```bash
# Create a cron job to rotate certificates before expiry
# /etc/cron.d/rke2-cert-rotation
0 2 1 */6 * root systemctl restart rke2-server

# Or use a Kubernetes CronJob to trigger rotation via a management script
```

---

## Best Practices

- Rotate certificates every 6 months rather than waiting for the 90-day auto-renewal window — this gives more predictable maintenance windows.
- Always take an etcd snapshot before rotating certificates — a failed rotation can leave the cluster in a broken state.
- Test certificate rotation in a staging cluster first to understand the restart sequence and any application impact.
