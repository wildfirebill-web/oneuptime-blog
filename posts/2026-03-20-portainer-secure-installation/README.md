# How to Secure Your Portainer Installation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Security, Docker, Hardening, DevOps

Description: Learn how to harden your Portainer installation with essential security controls including HTTPS, strong authentication, network restrictions, and access policies.

## Introduction

A default Portainer installation is functional but not hardened for production. This guide covers the essential security measures to protect your Portainer instance against unauthorized access, privilege escalation, and data exposure. Apply these measures before exposing Portainer to any network beyond localhost.

## Prerequisites

- Portainer CE or BE installed
- Admin access to the Portainer instance
- Access to the host system for network configuration
- A domain name and TLS certificate for HTTPS

## Security Checklist

- [ ] Enable HTTPS (TLS)
- [ ] Use a non-default admin username
- [ ] Set a strong admin password
- [ ] Restrict access by IP/VPN
- [ ] Disable telemetry
- [ ] Configure session timeout
- [ ] Restrict Docker socket access
- [ ] Enable RBAC for all users
- [ ] Disable unused features

## Step 1: Enable HTTPS

Never run Portainer over HTTP in production. Use HTTPS with a valid TLS certificate:

```bash
# Option A: Use Portainer with TLS certificates directly
docker run -d \
  --name portainer \
  --restart=unless-stopped \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /etc/portainer/certs:/certs \
  portainer/portainer-ce:latest \
  --ssl \
  --sslcert /certs/cert.pem \
  --sslkey /certs/key.pem

# Option B: Put Portainer behind a reverse proxy (Nginx/Traefik) that handles TLS
# See dedicated guides for Nginx and Traefik setups
```

## Step 2: Use a Non-Default Admin Username

During initial setup, change the admin username from `admin` to something less predictable:

```bash
# During initialization, use a custom username
curl -s -X POST https://portainer.example.com/api/users/admin/init \
  -H "Content-Type: application/json" \
  -d '{
    "username": "portainer-ops",
    "password": "$(openssl rand -base64 32)"
  }'
```

Or create a new admin user and remove the default `admin` user after setup.

## Step 3: Set a Strong Password

```bash
# Generate a cryptographically strong password
openssl rand -base64 32
# Example output: K8mP2xQzN9vR5tY3wE7jH1cL4uF6nD0a

# Or use Python
python3 -c "import secrets, string; print(secrets.token_urlsafe(32))"
```

Password requirements for Portainer admin:
- Minimum 12 characters
- Mix of upper/lowercase, numbers, and symbols
- Stored in a password manager or secrets vault
- Never shared via email or chat

## Step 4: Restrict Network Access

```bash
# Bind Portainer to a specific interface (not all interfaces)
docker run -d \
  --name portainer \
  -p 192.168.1.100:9443:9443 \  # Only accessible from internal IP
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Add firewall rules:

```bash
# UFW — Allow only specific source IPs
sudo ufw allow from 10.0.0.0/8 to any port 9443 comment "Portainer - VPN only"
sudo ufw deny 9443

# iptables
iptables -A INPUT -p tcp --dport 9443 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 9443 -j DROP
```

## Step 5: Configure Portainer Security Settings

In Portainer UI → **Settings** → **Security**:

1. **Session lifetime**: Set to `30m` or `1h` (not indefinite)
2. **Force HTTPS**: Enable (redirects HTTP to HTTPS)
3. **Disable telemetry**: Disable usage data collection
4. **Concurrent sessions**: Enable session limits

```bash
# Configure via API
TOKEN="your-admin-token"

curl -s -X PUT https://portainer.example.com/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "enableTelemetry": false,
    "authenticationMethod": 1,
    "userSessionTimeout": "30m",
    "enforceEdgeID": true
  }'
```

## Step 6: Enable RBAC for All Users

Never give users more access than they need:

1. Create teams aligned with job functions
2. Assign teams to specific environments with minimal required access
3. Use **Read-only** access for developers who only need to view logs/status
4. Use **Standard** access for users who deploy applications
5. Use **Administrator** access only for ops/infra team

```bash
# Create a read-only user via API
curl -s -X POST https://portainer.example.com/api/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"username": "readonly-dev", "password": "Pass123!", "role": 2}'

# Grant read-only access to production (role 3 = read-only)
curl -s -X PUT https://portainer.example.com/api/endpoints/1/useraccesspolicies \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"USER_ID": {"RoleId": 3}}'
```

## Step 7: Restrict Docker Dangerous Settings

In Portainer UI → **Environments** → select environment → **Security**:

- **Disable bind mounts** for non-admin users
- **Disable privileged mode** for non-admin users
- **Disable host PID/network** access for non-admin users
- **Disable container capabilities** addition for non-admin users
- **Enable read-only registry** restriction (require approval for external images)

## Step 8: Harden the Docker Socket

The Docker socket gives root-equivalent access. Restrict it:

```bash
# Portainer should be the only service with Docker socket access
# Use a dedicated Docker group and add only Portainer user
groupadd docker
usermod -aG docker portainer-user

# Set strict permissions on socket
chmod 660 /var/run/docker.sock
chown root:docker /var/run/docker.sock
```

## Step 9: Enable Container Content Trust

```bash
# Set environment variable for Portainer container
docker run -d \
  --name portainer \
  -e DOCKER_CONTENT_TRUST=1 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/portainer-ce:latest
```

## Step 10: Monitor and Audit

- Review **Authentication logs** (BE) regularly
- Set up log forwarding to a SIEM system
- Monitor for failed login attempts

```bash
# Check Portainer logs for suspicious activity
docker logs portainer 2>&1 | grep -E "(failed|unauthorized|error|blocked)" | tail -50
```

## Conclusion

Securing Portainer requires a layered approach: HTTPS for transport security, strong authentication for access control, network restrictions to limit exposure, and RBAC to minimize blast radius. Apply all steps before production deployment, and review the security settings periodically as your infrastructure grows. Portainer Business Edition provides additional security features including activity logs, SSO, and granular RBAC that further strengthen your security posture.
