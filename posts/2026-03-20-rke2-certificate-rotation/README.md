# How to Rotate RKE2 Certificates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Certificates, TLS, Security, Maintenance

Description: Learn how to rotate TLS certificates in RKE2 to maintain security and prevent certificate expiration from causing cluster outages.

Kubernetes uses TLS certificates extensively for secure communication between components. These certificates have expiration dates, and failing to rotate them can cause cluster outages when they expire. RKE2 provides built-in certificate rotation capabilities that simplify this critical maintenance task. This guide covers manual and automatic certificate rotation.

## Prerequisites

- RKE2 cluster running
- Root access to server nodes
- Understanding of when your certificates expire

## Understanding RKE2 Certificate Locations

RKE2 manages certificates for:
- etcd client and peer certificates
- API server certificates (client CA, service account key)
- kubelet client certificates
- kube-proxy certificates

Certificates are stored at: `/var/lib/rancher/rke2/server/tls/`

## Step 1: Check Certificate Expiration

```bash
# Check expiration of all RKE2 certificates
sudo find /var/lib/rancher/rke2/server/tls -name "*.crt" \
  -exec sh -c 'echo "=== {} ===" && \
  openssl x509 -in {} -noout -dates 2>/dev/null' \;

# Check specific important certificates
CERTS=(
  "/var/lib/rancher/rke2/server/tls/server-ca.crt"
  "/var/lib/rancher/rke2/server/tls/client-ca.crt"
  "/var/lib/rancher/rke2/server/tls/etcd/server-ca.crt"
)

for cert in "${CERTS[@]}"; do
  if [ -f "$cert" ]; then
    echo "=== $(basename $cert) ==="
    openssl x509 -in "$cert" -noout -dates
    # Check if expiring within 90 days
    openssl x509 -in "$cert" -noout -checkend 7776000 && \
      echo "OK: Not expiring in 90 days" || \
      echo "WARNING: Expiring within 90 days!"
    echo ""
  fi
done

# Check kubelet client certificate expiration
sudo find /var/lib/rancher/rke2/agent -name "*.crt" \
  -exec sh -c 'echo "=== {} ===" && \
  openssl x509 -in {} -noout -enddate 2>/dev/null' \;
```

## Step 2: Automatic Certificate Rotation

RKE2 supports automatic certificate rotation through the kubelet:

```yaml
# /etc/rancher/rke2/config.yaml - Enable automatic cert rotation
kubelet-arg:
  # Enable automatic rotation of kubelet client certificates
  - "rotate-certificates=true"

  # Enable automatic rotation of kubelet server certificates
  - "rotate-server-certificates=true"
```

```bash
# Apply the configuration
sudo systemctl restart rke2-server

# Verify kubelet certificate rotation is enabled
sudo ps aux | grep kubelet | tr ' ' '\n' | grep rotate
```

## Step 3: Manual Certificate Rotation

For rotating all cluster certificates:

```bash
# RKE2 provides a built-in certificate rotation command
# This rotates ALL certificates and requires a restart

# First, take an etcd snapshot for backup
sudo rke2 etcd-snapshot save \
  --name pre-cert-rotation-$(date +%Y%m%d-%H%M%S)

# Rotate all certificates
# This stops the RKE2 server, rotates certs, and restarts
sudo rke2 certificate rotate

# For HA clusters, run on each server node one at a time
# The command handles the rotation and restart automatically
```

## Step 4: Rotate Specific Certificates

```bash
# Rotate only specific certificate types
# Check RKE2 documentation for specific rotation options

# For custom rotation, you can:
# 1. Stop RKE2
sudo systemctl stop rke2-server

# 2. Remove the certificate files you want to rotate
# RKE2 will regenerate them on startup
# WARNING: Be careful about which certs to remove!
sudo rm /var/lib/rancher/rke2/server/tls/server-ca.crt 2>/dev/null
sudo rm /var/lib/rancher/rke2/server/tls/server-ca.key 2>/dev/null

# 3. Restart RKE2 (it will regenerate the removed certificates)
sudo systemctl start rke2-server

# 4. Monitor the certificate generation
sudo journalctl -u rke2-server -f | grep -i cert
```

## Step 5: Update kubeconfig After Rotation

After rotating certificates, kubeconfig files may need updating:

```bash
# After certificate rotation, update your kubeconfig
# The cluster CA may have changed

# Back up existing kubeconfig
cp ~/.kube/config ~/.kube/config.backup

# Get the new kubeconfig
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# If the server address is 127.0.0.1, update to actual IP/hostname
sed -i 's/127.0.0.1/<SERVER_IP>/' ~/.kube/config

# Test the connection
kubectl get nodes

# If nodes are not responding, check the certificate
kubectl get nodes 2>&1 | grep -i "certificate\|tls\|x509"
```

## Step 6: Update Agent Certificates

After rotating server certificates, agent nodes may need certificate updates:

```bash
# On each agent node, check if the agent can still connect
sudo journalctl -u rke2-agent | tail -20 | grep -E "error|Error|certificate"

# If agents show certificate errors, restart them
sudo systemctl restart rke2-agent

# Or if the CA has changed, you may need to remove the agent certs
# and let them re-register:
sudo systemctl stop rke2-agent

# Remove old agent certificates
sudo rm -rf /var/lib/rancher/rke2/agent/client-ca.crt
sudo rm -rf /var/lib/rancher/rke2/agent/server-ca.crt

# Restart agent to re-register with new certificates
sudo systemctl start rke2-agent

# Verify the agent reconnected
sudo journalctl -u rke2-agent -f | grep "node registered"
```

## Step 7: Monitor Certificate Expiration with Prometheus

```yaml
# prometheus-cert-alert.yaml - Alert before certificate expiration
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-cert-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: kubernetes-certificates
    rules:
    # Alert 90 days before expiration
    - alert: KubernetesClientCertificateExpiringSoon
      expr: |
        apiserver_client_certificate_expiration_seconds_count{job="apiserver"} > 0
        and histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m])))
        < 7776000  # 90 days in seconds
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Kubernetes client certificate expiring within 90 days"
        description: "Client certificate {{ $labels.job }} is expiring in less than 90 days"

    # Alert 30 days before expiration
    - alert: KubernetesClientCertificateExpiringCritical
      expr: |
        apiserver_client_certificate_expiration_seconds_count{job="apiserver"} > 0
        and histogram_quantile(0.01, sum by (job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m])))
        < 2592000  # 30 days in seconds
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Kubernetes client certificate expiring within 30 days"
```

## Conclusion

Certificate rotation is a critical maintenance task for Kubernetes clusters that is often overlooked until certificates expire and cause cluster outages. RKE2's built-in certificate rotation command simplifies this process significantly. For production clusters, implement Prometheus alerts for certificate expiration and schedule regular certificate rotation as part of your maintenance calendar. Enable automatic kubelet certificate rotation to handle the most common certificate type without manual intervention.
