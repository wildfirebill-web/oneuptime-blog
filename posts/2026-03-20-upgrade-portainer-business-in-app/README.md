# How to Upgrade Portainer Business Edition with In-App Updates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Business Edition, Upgrade, Docker, Maintenance

Description: Learn how to use Portainer Business Edition's built-in upgrade feature to update to the latest version directly from the UI.

---

Portainer Business Edition includes an in-app upgrade feature that lets administrators update Portainer without touching the command line. This feature automates the pull-stop-replace process from within the UI itself.

## Prerequisites

- Portainer BE running on Docker standalone
- Admin access to the Portainer UI
- Internet access from the Portainer host to Docker Hub

## Using the In-App Upgrade

### Step 1: Navigate to the Upgrade Page

1. Log in to Portainer as an administrator
2. Go to **Settings** in the left sidebar
3. Select **Upgrade** (or look for the upgrade notification banner)

### Step 2: Review the Available Update

Portainer checks for newer versions automatically. The upgrade page shows:
- Current version installed
- Latest available version
- Changelog summary

### Step 3: Trigger the Upgrade

Click **Upgrade** to start the process. Portainer will:
1. Pull the new Business Edition image in the background
2. Stop the current container
3. Start the new container with the same configuration
4. Redirect your browser to the updated UI

The process typically takes 30-60 seconds depending on network speed.

## Manual Upgrade via Command Line (Alternative)

If the in-app upgrade is unavailable or you prefer CLI:

```bash
# Step 1: Back up data
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/portainer_be_backup_$(date +%Y%m%d).tar.gz -C /data .

# Step 2: Stop and remove current container
docker stop portainer
docker container rm portainer

# Step 3: Pull latest BE image
docker pull portainer/portainer-ee:latest

# Step 4: Restart with same configuration
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:latest
```

## After the Upgrade

Verify the upgrade was successful:

```bash
# Confirm the new image version
docker inspect portainer --format '{{.Config.Image}}'
```

In the UI, navigate to **Settings > About** to confirm the updated version number and that your license is still active.

## License Persistence

Your Business Edition license is stored in the Portainer data volume. It persists across upgrades automatically. If the license screen appears unexpectedly after an upgrade, re-enter your license key.

---

*Integrate [OneUptime](https://oneuptime.com) to receive alerts if Portainer becomes unavailable after an upgrade.*
