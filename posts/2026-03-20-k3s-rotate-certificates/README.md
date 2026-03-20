# How to Rotate K3s Certificates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: K3s, Kubernetes, Security, TLS, Certificates, DevOps

Description: Learn how to rotate TLS certificates in K3s to maintain security compliance and prevent certificate expiration issues.

## Introduction

K3s automatically generates TLS certificates during installation that expire after one year (or 90 days if they will expire within 90 days on the next restart). Certificate rotation is a critical security practice that prevents unexpected cluster outages caused by expired certificates. This guide covers both automatic and manual certificate rotation in K3s.

## Understanding K3s Certificates

K3s generates several certificates stored in `/var/lib/rancher/k3s/server/tls/`:

- **CA certificates**: Root of trust for the cluster
- **API server certificates**: Secures the kube-apiserver
- **Controller manager certificates**: For the controller-manager component
- **Scheduler certificates**: For the kube-scheduler
- **etcd certificates**: Secures etcd communication
- **Kubelet certificates**: For each node's kubelet

## Check Certificate Expiration

Before rotating, check when your certificates expire:

```bash
# Check all K3s certificate expiration dates
for cert in /var/lib/rancher/k3s/server/tls/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in "$cert" -noout -dates 2>/dev/null || true
done

# Check specific certificate
openssl x509 -in /var/lib/rancher/k3s/server/tls/server-ca.crt \
  -noout -text | grep -A 2 "Validity"

# Quick expiry check (shows days until expiry)
openssl x509 -in /var/lib/rancher/k3s/server/tls/serving-kube-apiserver.crt \
  -noout -enddate
```

## Automatic Certificate Rotation

K3s automatically rotates certificates when it restarts if they will expire within **90 days**. Simply restarting K3s triggers this:

```bash
# Check if certificates need rotation (expiring within 90 days)
systemctl restart k3s

# K3s will automatically rotate certs that expire within 90 days
# Verify new expiration dates
for cert in /var/lib/rancher/k3s/server/tls/*.crt; do
  echo "$cert: $(openssl x509 -in $cert -noout -enddate 2>/dev/null)"
done
```

## Manual Certificate Rotation

To force certificate rotation regardless of expiration:

```bash
# Step 1: Stop the K3s service
systemctl stop k3s

# Step 2: Run certificate rotation
# This regenerates all certificates
k3s certificate rotate

# Step 3: Start K3s
systemctl start k3s

# Step 4: Verify K3s is healthy
kubectl get nodes
```

### Rotating Specific Certificates

You can also rotate only specific certificates:

```bash
# Stop K3s
systemctl stop k3s

# Rotate only the API server certificate
k3s certificate rotate --service kube-apiserver

# Rotate etcd certificates
k3s certificate rotate --service etcd

# Rotate all certificates explicitly
k3s certificate rotate \
  --service kube-apiserver \
  --service kube-scheduler \
  --service kube-controller-manager \
  --service k3s-controller \
  --service k3s-server \
  --service admin

# Start K3s
systemctl start k3s
```

## Rotating CA Certificates

CA rotation is more involved since all leaf certificates must be re-issued. K3s does not automatically rotate CA certificates — they have a 10-year validity period. If you need to rotate CAs:

```bash
# Step 1: Stop K3s on ALL nodes (servers and agents)
systemctl stop k3s        # On server nodes
systemctl stop k3s-agent  # On agent nodes

# Step 2: Back up existing TLS directory
cp -r /var/lib/rancher/k3s/server/tls /var/lib/rancher/k3s/server/tls.backup

# Step 3: Remove certificate directory to force regeneration
rm -rf /var/lib/rancher/k3s/server/tls

# Step 4: Start K3s server — it will regenerate all CAs and certs
systemctl start k3s

# Step 5: Retrieve the new node token (changed after CA rotation)
cat /var/lib/rancher/k3s/server/node-token

# Step 6: Re-join agent nodes with the new token
# On each agent node:
systemctl stop k3s-agent

# Update the token in the agent config
echo "K3S_TOKEN=<new-token>" > /etc/systemd/system/k3s-agent.service.env

systemctl daemon-reload
systemctl start k3s-agent
```

## Update kubeconfig After Rotation

After rotating certificates, update your local kubeconfig:

```bash
# Copy the new kubeconfig from the server
scp root@<server-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/config

# Update the server address if needed
sed -i 's/127.0.0.1/<server-ip>/g' ~/.kube/config

# Verify connectivity
kubectl cluster-info
kubectl get nodes
```

## Automate Certificate Rotation

Create a script to rotate certificates before they expire:

```bash
#!/bin/bash
# /usr/local/bin/k3s-cert-rotation-check.sh

DAYS_THRESHOLD=30
CERT_FILE="/var/lib/rancher/k3s/server/tls/serving-kube-apiserver.crt"

# Get days until expiry
EXPIRY=$(openssl x509 -in "$CERT_FILE" -noout -enddate | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
NOW_EPOCH=$(date +%s)
DAYS_LEFT=$(( (EXPIRY_EPOCH - NOW_EPOCH) / 86400 ))

echo "Certificate expires in $DAYS_LEFT days"

if [ "$DAYS_LEFT" -lt "$DAYS_THRESHOLD" ]; then
  echo "Rotating certificates (less than $DAYS_THRESHOLD days remaining)"
  systemctl stop k3s
  k3s certificate rotate
  systemctl start k3s
  echo "Certificate rotation complete"
fi
```

## Conclusion

Regular certificate rotation is a critical security hygiene practice. K3s makes it straightforward with built-in `certificate rotate` commands. For most clusters, the automatic rotation on restart is sufficient — but for production environments, implement proactive rotation checks and alerting before certificates approach expiration to avoid unexpected outages.
