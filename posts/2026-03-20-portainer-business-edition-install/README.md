# How to Install Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, portainer-business, installation, enterprise, license

Description: A guide to installing Portainer Business Edition with license activation, covering Docker, Docker Swarm, and Kubernetes deployments.

## Overview

Portainer Business Edition (BE) adds enterprise features on top of Portainer CE including multi-cluster management, RBAC with teams and roles, LDAP/AD integration, audit logging, Git integration for stacks, and automated backups. This guide covers installing Portainer BE and activating your license.

## Obtaining a Portainer Business License

1. Visit https://www.portainer.io/pricing
2. Choose a plan (Starter, Teams, or Business)
3. Purchase or start a free trial
4. You'll receive a license key via email

## Installation on Docker Standalone

```bash
# Create data volume
docker volume create portainer_data

# Install Portainer Business Edition
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest    # Note: portainer-ee for Business Edition

# Verify
docker ps | grep portainer
```

## Installation on Docker Swarm

```bash
# Deploy Portainer BE as a Docker Swarm service
docker service create \
  --name portainer \
  --constraint 'node.role == manager' \
  --publish mode=host,target=9443,published=9443 \
  --publish mode=host,target=8000,published=8000 \
  --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  --mount type=volume,src=portainer_data,dst=/data \
  --replicas=1 \
  portainer/portainer-ee:latest
```

## Installation on Kubernetes

```bash
# Install Portainer BE on Kubernetes
kubectl apply -f https://downloads.portainer.io/ee2-20/portainer.yaml

# Or via Helm
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

helm install portainer portainer/portainer \
  --namespace portainer \
  --create-namespace \
  --set enterpriseEdition.enabled=true \
  --set service.type=LoadBalancer \
  --set tls.force=true
```

## Activating Your License

### Via the Web UI

1. Access Portainer BE at `https://your-server:9443`
2. Complete the initial admin account setup
3. On the first screen, enter your license key
4. Click "Submit"

### Via Docker Run Command

```bash
# Pre-activate license during deployment
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest \
  --license-key="YOUR-LICENSE-KEY-HERE"
```

## Portainer BE vs CE Feature Comparison

| Feature | CE | BE Starter | BE Teams | BE Business |
|---|---|---|---|---|
| Price | Free | $5/node/mo | $10/node/mo | $25/node/mo |
| Nodes | Unlimited | 3 | 10 | Unlimited |
| RBAC | Basic | Advanced | Advanced | Advanced |
| Teams | No | No | Yes | Yes |
| LDAP/AD | No | No | Yes | Yes |
| Audit Logs | No | No | Yes | Yes |
| Git Integration (Stacks) | No | No | Yes | Yes |
| Automated Backups | No | No | No | Yes |
| Registry Management | Basic | Full | Full | Full |
| Multi-cluster | No | No | Yes | Yes |
| Support | Community | Email | Priority | Premium |

## Setting Up LDAP/AD Integration (BE Feature)

```
Portainer UI → Settings → Authentication → LDAP

Configuration:
- Server type: Active Directory (or OpenLDAP)
- Server URL: ldap://ad.company.com:389 (or ldaps for SSL)
- Service Account DN: CN=portainer-svc,OU=Service,DC=company,DC=com
- Password: [service account password]
- Base DN: DC=company,DC=com
- Username attribute: sAMAccountName
- Group membership attribute: memberOf
```

## Setting Up Team-Based Access (BE Feature)

```
Portainer UI → Settings → Teams → Add Team

Create teams:
- "DevOps Team" - maps to AD group "CN=k8s-admins,..."
- "Developers" - maps to AD group "CN=k8s-developers,..."
- "Read Only" - maps to AD group "CN=k8s-readonly,..."

Then assign teams to environments with appropriate roles.
```

## Configuring Automated Backups (BE Feature)

```
Portainer UI → Settings → Backup → Configure Backup

Options:
- Schedule: Cron expression (e.g., 0 2 * * * for 2 AM daily)
- Destination: S3, Azure Blob, or local file
- Retention: Number of backups to keep
- Encryption: Enable with a password
```

## Conclusion

Portainer Business Edition extends Portainer CE with enterprise features that are essential for team environments: RBAC with teams, LDAP/AD integration, audit logging, and automated backups. The installation process is nearly identical to Portainer CE, with the key difference being the `portainer-ee` image name and license activation. The Business Edition transforms Portainer from a personal management tool into an enterprise-ready container management platform.
