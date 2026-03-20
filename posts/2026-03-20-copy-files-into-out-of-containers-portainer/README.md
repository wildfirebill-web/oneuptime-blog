# How to Copy Files Into and Out of Containers in Portainer - Into Out

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, File Management, DevOps, Containers, Operations

Description: Use Portainer's file browser and Docker CLI to copy files into running containers and extract files from containers for debugging, configuration updates, and data export.

---

Sometimes you need to get files in or out of a running container - copy a configuration file, extract a log file, or update an asset. Portainer provides a built-in file browser for some operations, while `docker cp` is the primary tool for copying files.

## Method 1: Portainer's Built-in File Browser

For containers with accessible filesystems:

1. Go to **Containers > [Container Name]**
2. Click the **Files** tab (available when the container is running)
3. Browse the container filesystem
4. Use the upload/download buttons to transfer files

The Files tab provides a read-only browser with download capability. For uploads, use the container's console.

## Method 2: Docker cp (via Portainer Console or Host)

`docker cp` copies files between the host and container filesystem:

```bash
# Copy FROM container TO host

docker cp container_name:/path/inside/container /path/on/host

# Examples:
# Extract a log file from the container
docker cp webapp:/var/log/app/error.log ./error.log

# Extract the nginx configuration
docker cp nginx:/etc/nginx/nginx.conf ./nginx.conf

# Copy a directory out of the container
docker cp database:/var/backups ./database-backups/
```

```bash
# Copy FROM host TO container
docker cp /path/on/host container_name:/path/inside/container

# Examples:
# Update an app configuration without rebuilding
docker cp ./config.json webapp:/app/config.json

# Copy SSL certificates into a running nginx container
docker cp ./certs/ nginx:/etc/nginx/certs/

# Send a script into a container for one-time execution
docker cp ./fix-permissions.sh webapp:/tmp/fix-permissions.sh
docker exec webapp bash /tmp/fix-permissions.sh
```

## Method 3: Portainer Console

For smaller files, use the Portainer container console:

1. Open **Containers > [Container Name] > Console**
2. Use `cat` to view files
3. Use `echo` to create small files

```bash
# In the Portainer console
# View a configuration file
cat /app/config.json

# Create/update a small file
echo '{"debug": true, "logLevel": "verbose"}' > /app/config.json

# Download a file by encoding it (for environments without docker cp access)
base64 /app/config.json
# Copy the base64 output, decode it on your local machine
```

## Method 4: Volume Access

For files stored in named volumes, access them directly from the host:

```bash
# Named volumes are stored at /var/lib/docker/volumes/<volume-name>/_data/
ls /var/lib/docker/volumes/webapp-data/_data/

# Copy from volume (container can be stopped)
cp /var/lib/docker/volumes/webapp-data/_data/uploads ./uploads-backup

# Restore to volume
cp ./uploads-restore/* /var/lib/docker/volumes/webapp-data/_data/uploads/
```

## Use Cases

| Scenario | Recommended Method |
|----------|-------------------|
| Emergency config update | `docker cp` |
| Extract logs for debugging | `docker cp` |
| Browse container filesystem | Portainer Files tab |
| Update multiple files | Volume access |
| One-off script execution | Console + exec |

## Security Consideration

Copying files into containers bypasses the image build process and creates configuration drift - the container no longer matches its image. For persistent changes, rebuild the image or use volume mounts. Use direct file copying only for emergency fixes and debugging.

## Summary

Portainer's Files tab and `docker cp` provide practical ways to transfer files between the host and containers. For debugging and emergency fixes, these tools are invaluable. For production configuration management, prefer volume mounts or image rebuilds to maintain reproducibility.
