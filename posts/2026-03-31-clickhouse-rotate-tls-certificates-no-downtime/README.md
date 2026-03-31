# How to Rotate TLS Certificates in ClickHouse Without Downtime

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, TLS, Certificate, Security, Zero Downtime, Operations

Description: Learn how to rotate TLS certificates in ClickHouse without service interruption using reload-on-demand and graceful certificate replacement strategies.

---

TLS certificates expire and must be rotated periodically. ClickHouse supports certificate rotation without requiring a full restart, making it possible to update certificates with zero downtime in production environments.

## How ClickHouse Loads TLS Certificates

ClickHouse reads TLS certificates from disk when establishing new connections. It supports a `SYSTEM RELOAD TLS` command that forces the server to re-read certificate files without restarting. This is the key to zero-downtime rotation.

## Step 1: Generate the New Certificate

Generate a new certificate before replacing the old one:

```bash
# Generate new private key
openssl genrsa -out /tmp/server.key.new 4096

# Create CSR
openssl req -new \
  -key /tmp/server.key.new \
  -out /tmp/server.csr \
  -subj "/CN=clickhouse.example.com"

# Sign with your CA
openssl x509 -req -days 365 \
  -in /tmp/server.csr \
  -CA /etc/ssl/certs/ca.crt \
  -CAkey /etc/ssl/private/ca.key \
  -CAcreateserial \
  -out /tmp/server.crt.new
```

Verify the new certificate:

```bash
openssl x509 -in /tmp/server.crt.new -noout -text | grep -E "Not (Before|After)"
```

## Step 2: Replace Certificate Files Atomically

Copy the new files over the existing ones atomically to avoid a window where partial files could be read:

```bash
# Copy to temp location first
cp /tmp/server.key.new /etc/clickhouse-server/server.key.tmp
cp /tmp/server.crt.new /etc/clickhouse-server/server.crt.tmp

# Set permissions
chown clickhouse:clickhouse /etc/clickhouse-server/server.key.tmp /etc/clickhouse-server/server.crt.tmp
chmod 600 /etc/clickhouse-server/server.key.tmp

# Atomic rename
mv /etc/clickhouse-server/server.key.tmp /etc/clickhouse-server/server.key
mv /etc/clickhouse-server/server.crt.tmp /etc/clickhouse-server/server.crt
```

## Step 3: Reload TLS Without Restart

Tell ClickHouse to reload its TLS configuration:

```sql
SYSTEM RELOAD TLS;
```

Verify the new certificate is in use by checking its expiry:

```bash
echo | openssl s_client -connect localhost:9440 2>/dev/null \
  | openssl x509 -noout -dates
```

## Step 4: Verify Connections Still Work

Test that clients can connect using the new certificate:

```bash
clickhouse-client --secure --port 9440 --query "SELECT 1"
```

For HTTPS:

```bash
curl --cacert /etc/ssl/certs/ca.crt \
  -u default:password \
  'https://localhost:8443/?query=SELECT+1'
```

## Automating Certificate Rotation with cert-manager

If running in Kubernetes, use cert-manager annotations to auto-rotate and trigger a reload:

```bash
# After cert-manager renews, exec the reload command
kubectl exec -n clickhouse clickhouse-0 -- \
  clickhouse-client --query "SYSTEM RELOAD TLS"
```

## Rotating Interserver Certificates

For cluster deployments, rotate interserver certificates the same way, then reload on each node sequentially:

```bash
for node in ch-node-01 ch-node-02 ch-node-03; do
  ssh $node "mv /tmp/interserver.crt.new /etc/clickhouse-server/interserver.crt && \
             mv /tmp/interserver.key.new /etc/clickhouse-server/interserver.key"
  clickhouse-client --host $node --query "SYSTEM RELOAD TLS"
  echo "Reloaded TLS on $node"
done
```

## Monitoring Certificate Expiry

Set up a monitoring query or script to alert before expiry:

```bash
# Check days until expiry
openssl x509 -in /etc/clickhouse-server/server.crt -noout -enddate | \
  awk -F= '{print $2}' | \
  xargs -I{} date -d "{}" +%s | \
  xargs -I{} bash -c 'echo $(( ({} - $(date +%s)) / 86400 )) days remaining'
```

## Summary

Zero-downtime TLS certificate rotation in ClickHouse uses atomic file replacement followed by `SYSTEM RELOAD TLS`. This avoids service restarts while keeping certificates current. Automate rotation with certificate lifecycle tools and monitor expiry dates to prevent unexpected outages from expired certificates.
