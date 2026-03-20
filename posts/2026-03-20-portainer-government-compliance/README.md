# How to Use Portainer in Government and Compliance Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Government, FedRAMP, FISMA, Compliance

Description: Deploy and manage container infrastructure that meets government compliance requirements including FedRAMP, FISMA, and NIST 800-53 using Portainer.

## Introduction

Government agencies and contractors must comply with stringent frameworks including FedRAMP, FISMA, NIST SP 800-53, and STIG guidelines when deploying software systems. Container technology is increasingly adopted in government, but every component must meet compliance requirements. Portainer's audit capabilities, access controls, and deployment controls provide the foundation for compliant container operations.

## Compliance Framework Overview

| Framework | Key Controls | Portainer Feature |
|-----------|-------------|-------------------|
| NIST 800-53 AC-2 | Account management | LDAP/AD integration, user lifecycle |
| NIST 800-53 AC-3 | Access enforcement | RBAC, team namespace isolation |
| NIST 800-53 AU-2 | Audit events | Comprehensive activity logging |
| NIST 800-53 CM-7 | Least functionality | Read-only containers, restricted ports |
| STIG CAT I | No default passwords | Force password change on first login |

## Step 1: Air-Gapped Installation

Government environments often have no internet access:

```bash
# Download required images on an internet-connected system

docker pull portainer/portainer-ee:latest
docker pull portainer/agent:latest

# Save images to tar files
docker save portainer/portainer-ee:latest | gzip > portainer-ee-latest.tar.gz
docker save portainer/agent:latest | gzip > portainer-agent-latest.tar.gz

# Transfer to air-gapped environment (USB, classified transfer mechanism)
# On the air-gapped system:
docker load < portainer-ee-latest.tar.gz
docker load < portainer-agent-latest.tar.gz

# Run the pre-loaded image
docker run -d \
  --name portainer \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest \
  --no-analytics  # Disable telemetry in air-gapped environments
```

## Step 2: STIG-Compliant Docker Configuration

```bash
# /etc/docker/daemon.json - STIG hardening
cat > /etc/docker/daemon.json << 'EOF'
{
  "icc": false,
  "no-new-privileges": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userland-proxy": false,
  "live-restore": true,
  "userns-remap": "default",
  "seccomp-profile": "/etc/docker/seccomp-profile.json"
}
EOF

sudo systemctl restart docker
```

## Step 3: Configure FIPS-Compliant TLS

Government systems require FIPS 140-2 validated cryptography:

```bash
# Generate FIPS-compliant certificates
openssl req -x509 -newkey rsa:4096 \
  -keyout portainer.key \
  -out portainer.crt \
  -days 365 \
  -nodes \
  -subj "/CN=portainer.agency.gov" \
  -addext "subjectAltName=DNS:portainer.agency.gov"

# Configure Portainer with strong TLS settings
docker run -d \
  --name portainer \
  -p 9443:9443 \
  -v /certs:/certs:ro \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest \
  --ssl \
  --sslcert /certs/portainer.crt \
  --sslkey /certs/portainer.key \
  --http-disabled \
  --tlscacert /certs/ca.crt \
  --tlsverify
```

## Step 4: Configure CAC/PIV Authentication

Government users authenticate with Common Access Cards (CAC):

```bash
# Configure SAML with agency IdP (supports CAC/PIV via PKI)
# In Portainer: Settings > Authentication > SAML
# IdP Metadata URL: https://idp.agency.gov/metadata
# Entity ID: portainer.agency.gov

# For certificate-based auth, configure reverse proxy (nginx) with client cert validation
cat > /etc/nginx/conf.d/portainer.conf << 'EOF'
server {
    listen 443 ssl;
    server_name portainer.agency.gov;

    ssl_certificate /certs/server.crt;
    ssl_certificate_key /certs/server.key;
    ssl_client_certificate /certs/dod-root-ca.crt;
    ssl_verify_client on;
    ssl_verify_depth 10;

    # Extract Common Name from client cert as username
    proxy_set_header X-SSL-Client-CN $ssl_client_s_dn_cn;

    location / {
        proxy_pass https://localhost:9443;
    }
}
EOF
```

## Step 5: Implement NIST 800-53 Audit Controls

```bash
#!/bin/bash
# audit-export.sh - Export Portainer audit logs to SIEM
PORTAINER_URL="https://portainer.agency.gov"
API_KEY="audit-reader-token"
SIEM_URL="https://siem.agency.gov/api/logs"
SIEM_TOKEN="siem-ingestion-token"

# Fetch audit logs from Portainer
AUDIT_LOGS=$(curl -s \
  -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/useractivity?limit=500&offset=0")

# Format for SIEM ingestion
echo $AUDIT_LOGS | python3 -c "
import sys, json, datetime
logs = json.load(sys.stdin)
for entry in logs:
    event = {
        'source': 'portainer',
        'facility': 'container-management',
        'timestamp': entry.get('Timestamp', ''),
        'user': entry.get('Username', 'system'),
        'action': entry.get('Action', ''),
        'resource_type': entry.get('ResourceType', ''),
        'resource_id': entry.get('ResourceID', ''),
        'result': 'success',
        'classification': 'UNCLASSIFIED'
    }
    print(json.dumps(event))
" | curl -s -X POST \
    -H "Authorization: Bearer $SIEM_TOKEN" \
    -H "Content-Type: application/x-ndjson" \
    -d @- \
    "$SIEM_URL"

echo "Audit logs exported to SIEM"
```

## Step 6: Container STIG Compliance Scanning

```bash
#!/bin/bash
# stig-scan.sh - Check containers against STIG requirements
PORTAINER_URL="https://portainer.agency.gov"
API_KEY="scanner-api-key"

echo "=== Container STIG Compliance Scan ==="
echo "Date: $(date -u)"
echo ""

CONTAINERS=$(curl -s \
  -H "X-API-Key: $API_KEY" \
  "$PORTAINER_URL/api/endpoints/1/docker/containers/json?all=true")

python3 << 'PYTHON'
import json, sys

with open('/tmp/containers.json') as f:
    containers = json.load(f)

PASS = 0
FAIL = 0

for c in containers:
    name = c['Names'][0].lstrip('/') if c['Names'] else c['Id'][:12]

    # STIG Check: No privileged containers
    if c.get('HostConfig', {}).get('Privileged', False):
        print(f"CAT I FAIL: Privileged container: {name}")
        FAIL += 1
    else:
        PASS += 1

    # STIG Check: No host network mode
    if c.get('HostConfig', {}).get('NetworkMode') == 'host':
        print(f"CAT II FAIL: Host network mode: {name}")
        FAIL += 1

print(f"\nResults: {PASS} passed, {FAIL} failed")
PYTHON
```

## Step 7: Change Management and Configuration Control

```yaml
# Use GitOps with signed commits for change control
# .gitlab-ci.yml
stages:
  - security-scan
  - review
  - approve
  - deploy

security-scan:
  stage: security-scan
  script:
    - trivy image --exit-code 1 --severity CRITICAL $IMAGE_NAME

deployment-review:
  stage: review
  script:
    - echo "Manual review required for government deployment"
  when: manual
  allow_failure: false

approved-deploy:
  stage: deploy
  script:
    - curl -X POST "$PORTAINER_WEBHOOK_URL"
  when: manual
  needs: [deployment-review]
  only:
    - main
```

## Conclusion

Government container deployments require air-gapped installation capability, FIPS-validated cryptography, CAC/PIV authentication, comprehensive STIG-compliant configurations, and audit log forwarding to SIEM systems. Portainer Enterprise Edition provides the access control, activity logging, and team management that satisfy NIST 800-53 requirements. Combined with STIG-hardened Docker configurations and change management workflows, Portainer enables government agencies to leverage container technology within the bounds of their security frameworks.
