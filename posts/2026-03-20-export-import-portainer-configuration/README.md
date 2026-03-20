# How to Export and Import Portainer Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Configuration, Export, Import, Migration

Description: Learn how to export Portainer configuration for migration or backup purposes and import it into a new Portainer instance.

---

Exporting and importing Portainer configuration is useful when migrating to new hardware, cloning environments, or setting up disaster recovery. Portainer BE provides API endpoints specifically for this purpose.

## Export Configuration (Business Edition)

### Via the UI

1. Log in as an administrator
2. Navigate to **Settings > Backup & Restore**
3. Click **Download Backup**
4. Optionally set an encryption password
5. Save the downloaded `.tar.gz` file

### Via the API

```bash
# Authenticate and get JWT token
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Export the full Portainer configuration
curl -X POST \
  https://localhost:9443/api/backup \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"password":"export-encryption-key"}' \
  --output portainer_export_$(date +%Y%m%d).tar.gz \
  --insecure

echo "Export saved to: portainer_export_$(date +%Y%m%d).tar.gz"
```

## Export Configuration (Community Edition)

CE does not have a dedicated export API. Use volume backup instead:

```bash
# Stop Portainer for a consistent export
docker stop portainer

# Export the entire data volume
docker run --rm \
  -v portainer_data:/data \
  -v "$(pwd)":/export \
  alpine \
  tar czf /export/portainer_ce_export_$(date +%Y%m%d).tar.gz -C /data .

# Restart
docker start portainer

echo "CE export complete"
```

## Import Configuration (Business Edition)

### Via the UI

1. On a fresh Portainer BE installation, go to **Settings > Backup & Restore**
2. Click **Restore from file**
3. Select the backup `.tar.gz` file
4. Enter the decryption password if the export was encrypted
5. Click **Restore** — Portainer restarts with the imported configuration

### Via the API

```bash
# Import configuration on the target Portainer instance
curl -X POST \
  https://localhost:9443/api/restore \
  -H "Content-Type: multipart/form-data" \
  -F "file=@portainer_export_20260320.tar.gz" \
  -F "password=export-encryption-key" \
  --insecure
```

## Import Configuration (Community Edition)

```bash
# On the new server, create the volume and restore
docker volume create portainer_data

docker run --rm \
  -v portainer_data:/data \
  -v "$(pwd)":/import \
  alpine \
  tar xzf /import/portainer_ce_export_20260320.tar.gz -C /data

# Start Portainer with the restored data
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## What Is and Isn't Exported

| Exported | Not Exported |
|----------|-------------|
| Users and teams | Container runtime state |
| Environments/endpoints | Live container logs |
| Stacks configuration | Image data |
| Custom templates | Volume data |
| Access control policies | Secrets values (Swarm) |
| Settings and preferences | |

---

*Protect your Portainer deployment with proactive monitoring from [OneUptime](https://oneuptime.com).*
