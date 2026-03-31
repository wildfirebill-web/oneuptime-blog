# How to Rotate Cluster Certificates in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, Certificate

Description: Learn how to rotate internal Kubernetes cluster certificates in Rancher-managed RKE and RKE2 clusters to maintain security.

Kubernetes clusters use TLS certificates for internal communication between components such as the API server, kubelet, etcd, and controller manager. These certificates have expiration dates and must be rotated before they expire to prevent cluster outages. This guide covers rotating cluster certificates in Rancher-managed clusters.

## Prerequisites

- Rancher v2.5 or later
- RKE or RKE2 managed clusters
- Admin access to Rancher
- SSH access to cluster nodes (for manual operations)

## Step 1: Check Certificate Expiration

### Via Rancher UI

1. Navigate to the cluster.
2. In the cluster dashboard, check for certificate expiration warnings.
3. Rancher displays alerts when certificates are approaching expiration.

### Via kubectl

Check the API server certificate:

```bash
kubectl get configmap -n kube-system extension-apiserver-authentication \
  -o jsonpath='{.data.client-ca-file}' | openssl x509 -noout -dates
```

### On RKE2 Nodes

```bash
for cert in /var/lib/rancher/rke2/server/tls/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in "$cert" -noout -dates 2>/dev/null
done
```

### On RKE Nodes

```bash
for cert in /etc/kubernetes/ssl/*.pem; do
  echo "=== $cert ==="
  openssl x509 -in "$cert" -noout -dates 2>/dev/null
done
```

## Step 2: Rotate Certificates for RKE Clusters via Rancher

Rancher provides a one-click certificate rotation for RKE clusters:

1. Go to **Cluster Management**.
2. Click the three-dot menu on the RKE cluster.
3. Select **Rotate Certificates**.
4. Choose the rotation scope:
   - **Rotate all Service Certificates**: Rotates all internal cluster certificates.
   - **Rotate an Individual Service**: Rotate certificates for a specific component (etcd, kubelet, kube-apiserver, etc.).
5. Click **Rotate**.

Rancher will orchestrate the rotation across all nodes, restarting components as needed.

### Via the Rancher API

```bash
curl -X POST \
  'https://rancher.yourdomain.com/v3/clusters/CLUSTER_ID?action=rotateCertificates' \
  -H 'Authorization: Bearer YOUR_API_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "services": ["etcd", "kubelet", "kube-apiserver", "kube-controller-manager", "kube-scheduler", "kube-proxy"]
  }'
```

## Step 3: Rotate Certificates for RKE2 Clusters

RKE2 handles certificate rotation differently. On the control plane node:

### Automatic Rotation on Restart

RKE2 automatically rotates certificates that are within 90 days of expiry when the service is restarted:

```bash
systemctl restart rke2-server
```

### Force Certificate Rotation

To force rotation regardless of expiration date:

```bash
# Stop RKE2

systemctl stop rke2-server

# Delete the existing certificates (they will be regenerated)
rm -f /var/lib/rancher/rke2/server/tls/dynamic-cert.json

# Start RKE2
systemctl start rke2-server
```

### Rotate on Worker Nodes

On each worker node:

```bash
systemctl restart rke2-agent
```

The agent will automatically obtain new certificates from the server.

## Step 4: Rotate etcd Certificates

etcd certificates are critical for cluster data integrity:

### Via Rancher (RKE)

1. Go to **Cluster Management** > **Rotate Certificates**.
2. Select **etcd** as the service.
3. Click **Rotate**.

### Manually (RKE2)

```bash
systemctl stop rke2-server

# Back up existing etcd certs
cp -r /var/lib/rancher/rke2/server/tls/etcd /var/lib/rancher/rke2/server/tls/etcd.bak

# Remove etcd certs (they will be regenerated)
rm /var/lib/rancher/rke2/server/tls/etcd/server-client.crt
rm /var/lib/rancher/rke2/server/tls/etcd/server-client.key
rm /var/lib/rancher/rke2/server/tls/etcd/peer-server-client.crt
rm /var/lib/rancher/rke2/server/tls/etcd/peer-server-client.key

systemctl start rke2-server
```

## Step 5: Verify Rotated Certificates

After rotation, verify the new certificates:

```bash
# Check API server certificate
echo | openssl s_client -connect localhost:6443 2>/dev/null | \
  openssl x509 -noout -dates -subject

# Check etcd certificate
openssl x509 -in /var/lib/rancher/rke2/server/tls/etcd/server-client.crt \
  -noout -dates

# Check kubelet certificate
openssl x509 -in /var/lib/rancher/rke2/server/tls/client-kubelet.crt \
  -noout -dates
```

Verify cluster health after rotation:

```bash
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

## Step 6: Handle Multi-Node Rotation

For multi-node clusters, certificates should be rotated in a rolling fashion:

1. Rotate on the first control plane node.
2. Verify the node is healthy.
3. Rotate on the next control plane node.
4. Continue until all control plane nodes are done.
5. Rotate on worker nodes (can be done in parallel).

Monitor the cluster during rotation:

```bash
kubectl get nodes -w
```

## Step 7: Update kubeconfig Files

After certificate rotation, kubeconfig files may need to be regenerated:

### For RKE2

```bash
# The kubeconfig is automatically updated
cat /etc/rancher/rke2/rke2.yaml
```

### For Users

Users who have downloaded kubeconfig from Rancher should download a new copy:

1. In Rancher, navigate to the cluster.
2. Click **Kubeconfig File**.
3. Download the updated kubeconfig.

## Step 8: Set Up Certificate Monitoring

Monitor certificate expiration across all clusters:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cluster-cert-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: cluster-certificates
    rules:
    - alert: ClusterCertExpiring
      expr: |
        apiserver_client_certificate_expiration_seconds_count > 0
        and
        histogram_quantile(0.01, rate(apiserver_client_certificate_expiration_seconds_bucket[5m])) < 2592000
      labels:
        severity: warning
      annotations:
        summary: "Cluster certificates expiring within 30 days"
```

Create a script to check certificates across all clusters:

```bash
#!/bin/bash
for node in $(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'); do
  echo "=== Node: $node ==="
  ssh $node 'for cert in /var/lib/rancher/rke2/server/tls/*.crt; do
    expiry=$(openssl x509 -in "$cert" -noout -enddate 2>/dev/null | cut -d= -f2)
    echo "$cert: $expiry"
  done' 2>/dev/null
done
```

## Troubleshooting

### Cluster Becomes Unavailable After Rotation

If a node fails to rejoin after certificate rotation:

```bash
# Check kubelet logs
journalctl -u rke2-server -f
# or
journalctl -u rke2-agent -f
```

Restart the node's RKE2 service to trigger certificate re-negotiation.

### API Server Returns Certificate Errors

Clear the dynamic certificate cache:

```bash
rm /var/lib/rancher/rke2/server/tls/dynamic-cert.json
systemctl restart rke2-server
```

### kubectl Returns Authentication Errors

Download a fresh kubeconfig from Rancher after certificate rotation.

## Conclusion

Regular certificate rotation is essential for maintaining the security and availability of your Rancher-managed clusters. By using Rancher's built-in rotation features for RKE clusters and RKE2's automatic rotation on restart, you can keep certificates current with minimal disruption. Combine rotation with monitoring to prevent unexpected expirations.
