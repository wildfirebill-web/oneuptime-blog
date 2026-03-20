# How to Upgrade Portainer Business Edition with In-App Updates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Portainer-business, In-app-updates, Upgrade

Description: A guide to using Portainer Business Edition's in-app update feature for streamlined upgrades directly from the Portainer web interface.

## Overview

Portainer Business Edition includes an in-app update mechanism that simplifies upgrades by allowing administrators to trigger updates directly from the Portainer web interface. This guide covers using the in-app update feature and when to use it vs manual upgrade methods.

## Accessing In-App Updates

The in-app update feature is available in Portainer Business Edition:

```text
Portainer UI → Settings → Upgrade Portainer
```

If a new version is available, Portainer displays an update notification and a button to trigger the upgrade.

## Step 1: Check for Updates

```text
Portainer UI → Help → About → Check for Updates
```

Or look for the notification badge on the navigation menu.

## Step 2: Review Release Notes

Before upgrading, always review the release notes:

```text
1. Click on the version number in the update notification
2. Review breaking changes, new features, and bug fixes
3. Check if any configuration changes are required
```

## Step 3: Backup Before Updating

Even with in-app updates, backup first:

```bash
# Create backup via Portainer UI

Settings → Backup → Download backup

# Or via CLI
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/portainer-before-update-$(date +%Y%m%d).tar.gz -C /data .
```

## Step 4: Trigger the In-App Update

```text
Settings → Update & Restore → Update Portainer
→ Click "Update" button
→ Confirm the update in the dialog
→ Portainer will pull the new image, stop itself, and restart with the new version
```

The update process takes approximately 1-3 minutes. You will lose the browser connection while Portainer restarts - this is normal.

## Step 5: Verify the Update

After Portainer restarts, navigate back to `https://portainer:9443`:

```text
Help → About → Version should show the new version number
```

## When to Use In-App Updates vs Manual Upgrade

| Scenario | Recommended Method |
|---|---|
| Portainer BE, non-critical | In-App Update |
| Portainer BE, production | Manual upgrade with backup |
| Portainer CE | Manual docker stop/rm/run |
| Kubernetes deployment | Helm upgrade |
| Swarm deployment | Docker service update |
| Major version upgrade | Manual with careful review |

## Rollback After In-App Update

If the in-app update causes issues:

```bash
# Restore from backup
docker stop portainer && docker rm portainer

# Restore data volume from backup
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/portainer-before-update-$(date +%Y%m%d).tar.gz"

# Start with the previous version
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ee:2.19.0   # Previous version
```

## Automated Update Notifications

Configure Portainer BE to notify your team when updates are available:

```text
Settings → Notifications
→ Enable email notifications for "New Portainer version available"
→ Enter recipient email addresses
```

## Conclusion

Portainer Business Edition's in-app update feature simplifies the upgrade process for most deployments. For non-production environments, the in-app update provides a convenient one-click upgrade experience. For production environments, the best practice remains to backup first, then use the in-app update or manual process. The brief downtime during the update is typically 1-3 minutes, making it suitable for planned maintenance windows.
