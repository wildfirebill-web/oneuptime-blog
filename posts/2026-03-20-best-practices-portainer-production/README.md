# Best Practices for Running Portainer in Production

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Production, Best Practices, Security, High Availability, DevOps

Description: Run Portainer reliably in production with TLS termination, high availability, proper resource allocation, monitoring, and security hardening for enterprise-grade container management.

---

Running Portainer in production requires more than the basic quick-start. This guide covers TLS, high availability, resource sizing, monitoring, and security hardening for a production-grade Portainer deployment.

## Production Deployment Checklist

- [ ] TLS/HTTPS configured
- [ ] Strong admin password set
- [ ] LDAP/SSO authentication enabled
- [ ] Resource limits configured
- [ ] Monitoring and alerting set up
- [ ] Backup schedule configured
- [ ] Reverse proxy configured
- [ ] Firewall rules applied

## Step 1: TLS Configuration

Never run Portainer in production over HTTP. Use one of these approaches:

**Option A: Portainer with self-managed certificates:**

```bash
docker run -d \
  --name portainer \
  --restart always \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /opt/portainer/ssl:/certs \
  portainer/portainer-ee:latest \
  --sslcert /certs/portainer.crt \
  --sslkey /certs/portainer.key
```

**Option B: Nginx reverse proxy (recommended):**

```yaml
# portainer-production-stack.yml

version: "3.8"
services:
  portainer:
    image: portainer/portainer-ee:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    # Do NOT expose ports - nginx handles TLS termination
    restart: always
    networks:
      - portainer-internal

  nginx:
    image: nginx:1.25-alpine
    volumes:
      - /opt/nginx/portainer.conf:/etc/nginx/conf.d/portainer.conf:ro
      - /opt/certs:/etc/nginx/certs:ro
    ports:
      - "443:443"
    restart: always
    networks:
      - portainer-internal

networks:
  portainer-internal:
    driver: bridge
```

```nginx
# /opt/nginx/portainer.conf
server {
    listen 443 ssl;
    server_name portainer.example.com;
    
    ssl_certificate /etc/nginx/certs/portainer.crt;
    ssl_certificate_key /etc/nginx/certs/portainer.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    location / {
        proxy_pass http://portainer:9000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## Step 2: Resource Sizing

Portainer's resource requirements scale with the number of environments and users:

| Scale | Environments | Concurrent Users | Recommended RAM | CPU |
|-------|-------------|------------------|----------------|-----|
| Small | 1-10 | 1-5 | 512MB | 0.5 core |
| Medium | 10-50 | 5-20 | 1GB | 1 core |
| Large | 50+ | 20+ | 2GB | 2 cores |

Set resource limits:

```bash
docker run -d --name portainer \
  --memory="1g" \
  --cpus="1.0" \
  ...
```

## Step 3: Monitoring Portainer

Monitor Portainer's own health:

```yaml
  # Uptime Kuma or similar for Portainer availability monitoring
  uptime-kuma:
    image: louislam/uptime-kuma:1
    volumes:
      - uptime-kuma-data:/app/data
    ports:
      - "3001:3001"
    restart: always
```

Configure a monitor for `https://portainer.example.com/api/status` - this endpoint returns Portainer's operational status.

## Step 4: Security Hardening

```bash
# 1. Change default admin username (do this on first login)
# 2. Enable 2FA for admin accounts (BE feature)
# 3. Configure IP allowlist for Portainer access
# 4. Disable anonymous usage telemetry in Settings > General

# Firewall: only allow HTTPS from trusted IPs to Portainer
ufw allow from 10.0.0.0/8 to any port 443
ufw deny 443
```

## Step 5: High Availability

For production environments where Portainer downtime is unacceptable:

1. Run Portainer behind a load balancer
2. Use shared storage (NFS, S3-compatible) for the Portainer data volume
3. Configure health checks and auto-restart

Note: Portainer does not currently support active-active HA - this is a warm standby configuration.

## Step 6: Regular Maintenance

- **Update Portainer** - run updates monthly, following the upgrade guide
- **Prune unused resources** - schedule weekly `docker system prune` on all hosts
- **Review access** - quarterly user and permission review
- **Test backups** - monthly restore test from backup

## Summary

Production Portainer deployments require TLS, proper resource sizing, monitoring, security hardening, and a backup/restore plan. Treat Portainer as a critical piece of infrastructure - its availability affects all the services it manages.
