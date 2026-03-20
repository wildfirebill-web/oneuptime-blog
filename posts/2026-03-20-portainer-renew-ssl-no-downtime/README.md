# How to Renew SSL Certificates for Portainer Without Downtime

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, ssl, tls, certificate-renewal, zero-downtime, maintenance

Description: A guide to renewing SSL/TLS certificates for Portainer with minimal or zero downtime using proper renewal procedures.

## Overview

Certificate expiry causes immediate service disruption — browsers refuse connections and automation breaks. Portainer does not perform hot-reloading of certificates, so a container restart is required after renewal. However, with proper preparation and scripting, renewal can be completed in seconds with minimal user impact. This guide covers renewal strategies for all Portainer deployment types.

## Prerequisites

- Running Portainer with SSL configured
- Access to current certificate files
- Docker CLI access

## Understanding Portainer's Certificate Loading

Portainer loads SSL certificates only at startup. To apply a new certificate, you must restart the Portainer container. A restart typically takes 10-30 seconds, during which the UI is briefly unavailable.

## Strategy 1: Certificate Pre-Staging (Minimal Downtime)

Pre-stage the new certificate before restarting:

```bash
#!/bin/bash
# renew-portainer-cert.sh
set -e

DOMAIN="portainer.example.com"
CERT_PATH="/etc/letsencrypt/live/${DOMAIN}"
BACKUP_DIR="/opt/portainer-cert-backups/$(date +%Y%m%d-%H%M%S)"

echo "=== Portainer Certificate Renewal ==="
echo "1. Backing up current certificates..."
mkdir -p "${BACKUP_DIR}"
docker run --rm \
  -v portainer_data:/data \
  -v "${BACKUP_DIR}:/backup" \
  alpine \
  sh -c "cp /data/certs/cert.pem /backup/ && cp /data/certs/key.pem /backup/ 2>/dev/null; echo 'Backup complete'"

echo "2. Renewing certificate via Certbot..."
certbot renew --cert-name "${DOMAIN}" --quiet

echo "3. Staging new certificate into volume..."
docker run --rm \
  -v portainer_data:/data \
  -v "${CERT_PATH}:/certs:ro" \
  alpine \
  sh -c "mkdir -p /data/certs && \
    cp /certs/fullchain.pem /data/certs/cert.pem && \
    cp /certs/privkey.pem /data/certs/key.pem && \
    chmod 644 /data/certs/cert.pem && \
    chmod 600 /data/certs/key.pem"

echo "4. Restarting Portainer..."
docker restart portainer

echo "5. Waiting for Portainer to be ready..."
for i in $(seq 1 30); do
  if curl -sk https://localhost:9443/api/status | grep -q "Version"; then
    echo "Portainer is ready!"
    break
  fi
  sleep 1
  echo "  Waiting... (${i}/30)"
done

echo "6. Verifying new certificate..."
NEW_EXPIRY=$(echo | openssl s_client -connect localhost:9443 2>/dev/null \
  | openssl x509 -noout -enddate | cut -d= -f2)
echo "New certificate expiry: ${NEW_EXPIRY}"
echo "=== Renewal complete ==="
```

## Strategy 2: Rolling Update with Nginx (Zero Downtime)

If Portainer is behind Nginx, swap the proxy target during renewal:

```bash
#!/bin/bash
# zero-downtime-renew.sh

# 1. Start second Portainer instance on different port with new cert
docker run -d \
  -p 9444:9443 \
  --name portainer-new \
  --restart=no \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /etc/letsencrypt/live/portainer.example.com/fullchain.pem:/new-certs/cert.pem:ro \
  -v /etc/letsencrypt/live/portainer.example.com/privkey.pem:/new-certs/key.pem:ro \
  portainer/portainer-ce:latest \
  --ssl \
  --sslcert /new-certs/cert.pem \
  --sslkey /new-certs/key.pem

# 2. Wait for new instance to be ready
sleep 15
curl -sk https://localhost:9444/api/status

# 3. Swap Nginx to point to new instance
sed -i 's/proxy_pass https:\/\/localhost:9443/proxy_pass https:\/\/localhost:9444/' \
  /etc/nginx/conf.d/portainer.conf
nginx -s reload

# 4. Stop old Portainer
docker stop portainer
docker rm portainer

# 5. Rename new to old
docker rename portainer-new portainer

# 6. Restore Nginx config to standard port
sed -i 's/proxy_pass https:\/\/localhost:9444/proxy_pass https:\/\/localhost:9443/' \
  /etc/nginx/conf.d/portainer.conf
nginx -s reload
```

## Strategy 3: Scheduled Maintenance Window

For simplest operations, schedule during low-traffic hours:

```bash
# Add to crontab: renew at 3 AM on the 1st of each month
0 3 1 * * /usr/local/bin/renew-portainer-cert.sh >> /var/log/portainer-cert-renewal.log 2>&1
```

## Monitoring Certificate Expiry

```bash
#!/bin/bash
# check-cert-expiry.sh
EXPIRY=$(echo | openssl s_client -connect localhost:9443 2>/dev/null \
  | openssl x509 -noout -enddate | cut -d= -f2)
EXPIRY_TS=$(date -d "${EXPIRY}" +%s)
NOW_TS=$(date +%s)
DAYS_LEFT=$(( (EXPIRY_TS - NOW_TS) / 86400 ))

if [ "${DAYS_LEFT}" -lt 14 ]; then
  echo "WARNING: Portainer certificate expires in ${DAYS_LEFT} days!"
  # Send alert
fi
echo "Certificate expires in ${DAYS_LEFT} days (${EXPIRY})"
```

```bash
# Add to crontab: daily check
0 9 * * * /usr/local/bin/check-cert-expiry.sh
```

## Rollback Procedure

```bash
# If renewal fails, restore from backup
docker stop portainer

docker run --rm \
  -v portainer_data:/data \
  -v "${BACKUP_DIR}:/backup" \
  alpine \
  sh -c "cp /backup/cert.pem /data/certs/ && cp /backup/key.pem /data/certs/"

docker start portainer
echo "Rolled back to previous certificate"
```

## Conclusion

Portainer certificate renewal requires a container restart, but with proper scripting this takes under 30 seconds. The pre-staging approach (copy new cert, then restart) minimizes downtime. For truly zero-downtime renewals, use a Nginx reverse proxy and the rolling update approach. Always maintain certificate backups before renewal and test the renewal process before certificates actually expire.
