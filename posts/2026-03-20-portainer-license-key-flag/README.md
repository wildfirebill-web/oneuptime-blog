# How to Use the --license-key Flag for Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Business Edition, License, CLI, Configuration

Description: Apply and manage your Portainer Business Edition license key using the --license-key flag and UI settings to unlock enterprise features.

## Introduction

Portainer Business Edition (BE) requires a license key to activate enterprise features including RBAC, LDAP/SSO authentication, activity logging, scheduled backups, and advanced Kubernetes management. This guide shows how to apply the license key at startup and manage it afterward.

## Obtaining a License Key

1. Purchase Portainer Business Edition at [portainer.io](https://www.portainer.io/pricing)
2. Log in to the Portainer customer portal
3. Navigate to **License Keys**
4. Copy your license key (format: `XxXxXx-XXXX-XXXX-XXXX-XXXXXXXXXXXX`)

Free 5-node trial licenses are available without payment at [portainer.io/take-3](https://www.portainer.io/take-3).

## Step 1: Apply License at Startup

```bash
# Start Portainer BE with license key
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest \
  --license-key="XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"

# Note: Use portainer-ee (Enterprise Edition) image, not portainer-ce
```

## Step 2: Apply License via Environment Variable

For more secure handling (keeps key out of process list):

```bash
# Set license as environment variable
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -e PORTAINER_LICENSE_KEY="XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest
```

## Step 3: Apply License via Portainer UI

If Portainer is already running without a license:

1. Log in to Portainer
2. Go to **Settings** → **License**
3. Click **Apply License**
4. Enter your license key
5. Click **Submit**

## Step 4: Apply License via Docker Compose

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ee:latest
    container_name: portainer
    restart: unless-stopped
    command: --license-key=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

Or use a secret for better security:

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ee:latest
    container_name: portainer
    restart: unless-stopped
    environment:
      # Load from .env file: echo "PORTAINER_LICENSE=XXX" > .env
      PORTAINER_LICENSE_KEY: "${PORTAINER_LICENSE_KEY}"
    ports:
      - "9000:9000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
```

## Step 5: Verify License Activation

```bash
# Check via Portainer API
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/system/info | jq '{
  Version: .Version,
  InstanceID: .InstanceID
}'

# Or check license status
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/licenses | jq .

# In the UI: Settings → License → shows expiry date and node count
```

## Step 6: Update an Expired License

```bash
# Stop Portainer
docker stop portainer && docker rm portainer

# Apply new license key
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest \
  --license-key="NEW-LICENSE-KEY-XXXX-XXXX-XXXX-XXXXXXXXXXXX"

# Or update via API without restart
curl -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:9000/api/licenses \
  -d '{"key": "NEW-LICENSE-KEY-XXXX-XXXX-XXXX-XXXXXXXXXXXX"}'
```

## Step 7: Migrate License to New Portainer Instance

```bash
# Export license info from old instance
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:9000/api/licenses | jq '.key'

# Apply to new instance
docker run -d \
  -p 9000:9000 \
  --name portainer-new \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data_new:/data \
  portainer/portainer-ee:latest \
  --license-key="YOUR-LICENSE-KEY"
```

## Step 8: Use License Key File

```bash
# Create a license key file
echo "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX" > /opt/portainer/license.key
chmod 600 /opt/portainer/license.key

# Mount and reference it
docker run -d \
  -p 9000:9000 \
  --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /opt/portainer/license.key:/license.key:ro \
  portainer/portainer-ee:latest \
  --license-key="$(cat /opt/portainer/license.key)"

# Or in a script:
LICENSE=$(cat /opt/portainer/license.key)
docker run -d \
  --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest \
  --license-key="$LICENSE"
```

## Step 9: Check Features Unlocked by License

After applying the license, verify BE features are available:

1. **RBAC**: Settings → Users → Teams (should show team options)
2. **Activity Logs**: Settings → Activity Logs (should be accessible)
3. **Scheduled Backups**: Settings → Backup → S3 backup option visible
4. **LDAP/SSO**: Settings → Authentication → OAuth/LDAP options

## Conclusion

Applying the Portainer Business Edition license key via `--license-key` at startup is the most reliable method for automated deployments. For production environments, store the license key in an environment variable or Docker secret rather than passing it directly on the command line. The license unlocks enterprise features including RBAC, LDAP/OAuth SSO, activity auditing, and scheduled backups to S3.
