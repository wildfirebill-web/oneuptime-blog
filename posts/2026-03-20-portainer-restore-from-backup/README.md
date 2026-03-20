# How to Restore Portainer from a Backup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Restore, Disaster-recovery, Backup

Description: A guide to restoring Portainer from a backup after data loss, corruption, or failed upgrades, covering both manual and API-based restoration methods.

## Overview

Being able to restore Portainer from a backup is just as important as creating the backup. This guide covers how to restore Portainer CE and Business Edition from various backup types including volume tar archives, database file copies, and Portainer BE's native backup format.

## Restore from Docker Volume Backup (tar archive)

This is the most common restore scenario:

```bash
# Step 1: Stop and remove current Portainer container

docker stop portainer 2>/dev/null || true
docker rm portainer 2>/dev/null || true

# Step 2: Remove corrupted/old data volume (or rename it)
docker volume rm portainer_data 2>/dev/null || true
docker volume create portainer_data

# Step 3: Restore data from backup
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine \
  sh -c "cd /data && tar xzf /backup/portainer-backup-20260319.tar.gz"

# Verify restoration
docker run --rm \
  -v portainer_data:/data \
  alpine ls -la /data

# Step 4: Start Portainer with restored data
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

echo "Portainer restored successfully"
```

## Restore from BoltDB File Copy

```bash
# Stop Portainer
docker stop portainer

# Replace the database file
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine cp /backup/portainer-20260319.db /data/portainer.db

# Start Portainer
docker start portainer
```

## Restore Portainer BE from Native Backup

```bash
# Via the Portainer UI (after logging in as admin)
# Settings → Backup → Restore from backup → Upload ZIP file
# Note: Restoring via UI requires Portainer to be running

# Via API
TOKEN=$(curl -s -k \
  -X POST https://portainer:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"currentPassword"}' \
  | jq -r '.jwt')

curl -s -k \
  -X POST https://portainer:9443/api/restore \
  -H "Authorization: Bearer ${TOKEN}" \
  -F "file=@portainer-backup-20260319.zip" \
  -F "password=encryption-password"
```

## Restore on Kubernetes

```bash
# Scale down Portainer deployment first
kubectl scale deployment portainer --replicas=0 -n portainer

# Wait for pod to terminate
kubectl wait --for=delete pod -l app=portainer -n portainer --timeout=60s

# Restore data to PVC using a Job
kubectl apply -f - << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: portainer-restore
  namespace: portainer
spec:
  template:
    spec:
      initContainers:
        - name: restore
          image: alpine
          command:
            - sh
            - -c
            - |
              cd /data && tar xzf /backup/portainer-backup.tar.gz
              echo "Restore complete"
          volumeMounts:
            - name: portainer-data
              mountPath: /data
            - name: backup
              mountPath: /backup
      containers:
        - name: done
          image: alpine
          command: ["echo", "Restore job complete"]
      volumes:
        - name: portainer-data
          persistentVolumeClaim:
            claimName: portainer
        - name: backup
          configMap:
            name: portainer-backup   # Pre-load backup into ConfigMap
      restartPolicy: Never
EOF

# Scale Portainer back up
kubectl scale deployment portainer --replicas=1 -n portainer
```

## Post-Restore Verification

```bash
# Check Portainer is running
docker ps | grep portainer

# Check logs for errors
docker logs portainer --tail=20

# Verify data is restored
# Log in to Portainer UI and verify:
# - Environments are configured
# - Users are present
# - Stacks are visible (though they may need reconnecting)
```

## Troubleshooting Restore Issues

### Database Version Mismatch

If you restore to an older Portainer version's backup on a newer version:

```bash
# Start Portainer with -no-analytics flag to ensure clean startup
docker run -d \
  ...
  portainer/portainer-ce:2.19.0 \   # Use the version matching your backup
  --no-analytics
```

### Corrupted Backup

```bash
# Verify backup integrity before restoring
tar tzf portainer-backup.tar.gz | head
# If this shows files, backup is valid
# If error, backup is corrupted - use another backup
```

## Conclusion

Restoring Portainer from a backup is straightforward when you follow the correct steps. The key is to stop Portainer before restoring data to avoid corruption, restore to a fresh volume, and verify the restoration by logging in and checking your configuration. Always test your restore procedure before a real incident - restore to a test environment periodically to confirm backups are working correctly.
