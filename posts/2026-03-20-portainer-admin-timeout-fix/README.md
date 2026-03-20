# How to Fix the 5-Minute Admin Timeout in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Troubleshooting, Self-Hosted, Security

Description: Learn why Portainer locks itself after 5 minutes without admin account creation and how to properly reset or bypass this timeout.

## Introduction

When you first install Portainer, you have exactly 5 minutes to navigate to the UI and create an admin account. If that window expires, Portainer displays an error and refuses all access. This security feature prevents unauthorized users from claiming your uninitialized instance - but it can catch new users off guard.

## What Happens After the Timeout

After 5 minutes without initialization:
- The UI shows: *"Your Portainer instance timed out for security purposes."*
- Port 9000 and 9443 remain accessible, but no login or setup is possible
- The container continues running but is effectively locked

## Method 1: Reset via --admin-password Flag (Recommended)

The cleanest approach is to provide the admin password at startup:

```bash
# Stop and remove the existing container (keep or remove the volume)

docker stop portainer
docker rm portainer

# Option A: Inline password (less secure, visible in process list)
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --admin-password='$2y$05$8qOfvkl7D4FtcC/eCIbVGeFNQtYjC6.gg5bflnEsOxOinqPgXHzaC'

# The hash above is bcrypt for "admin123" - generate your own:
# docker run --rm httpd:2.4-alpine htpasswd -nbB admin yourpassword | cut -d ':' -f 2

# Option B: Password from environment variable
HASHED_PASS=$(docker run --rm httpd:2.4-alpine htpasswd -nbB admin yourpassword | cut -d ':' -f 2)

docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --admin-password="$HASHED_PASS"
```

## Method 2: Reset by Removing the Data Volume

If you haven't configured anything in Portainer yet:

```bash
# Stop the timed-out container
docker stop portainer
docker rm portainer

# Remove the old data volume
docker volume rm portainer_data

# Start fresh - you'll have a new 5-minute window
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Navigate to http://your-host:9000 IMMEDIATELY
```

## Method 3: Delete the portainer.db File

If you've already configured some settings in Portainer and don't want to lose them, you can selectively delete only the initialization flag:

```bash
# Access the Portainer data volume
docker run --rm -it \
  -v portainer_data:/data \
  alpine:latest \
  sh -c "ls -la /data/"

# The portainer.db is a BoltDB file
# Remove only the database to reset initialization
docker run --rm \
  -v portainer_data:/data \
  alpine:latest \
  rm /data/portainer.db

# Restart Portainer - you'll have a fresh 5-minute window
docker restart portainer
```

> **Warning**: Removing `portainer.db` deletes all Portainer configuration (environments, users, stacks metadata). Backed-up stacks and containers are unaffected.

## Method 4: Use --admin-password-file Flag

For more secure password handling:

```bash
# Create a password file
echo "yourpassword" > /tmp/portainer-password
chmod 600 /tmp/portainer-password

# Generate the bcrypt hash
HASH=$(docker run --rm -v /tmp/portainer-password:/pwd \
  httpd:2.4-alpine \
  sh -c "htpasswd -nbB admin \$(cat /pwd)" | cut -d ':' -f 2)

echo "$HASH" > /tmp/portainer-hash

# Start Portainer with the password file
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /tmp/portainer-hash:/tmp/portainer-hash:ro \
  portainer/portainer-ce:latest \
  --admin-password-file=/tmp/portainer-hash
```

## Preventing Future Timeouts

Use an automation script that starts Portainer AND immediately sets up the admin account:

```bash
#!/bin/bash
# deploy-portainer.sh

# Start Portainer
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --admin-password-file=/run/secrets/portainer-password

echo "Portainer started. Admin account configured via password file."
echo "Access: http://$(hostname -I | awk '{print $1}'):9000"
```

## Using Docker Compose

In a compose file, you can avoid the timeout entirely:

```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    command: >
      --admin-password=$$2y$$05$$8qOfvkl7D4FtcC/eCIbVGeFNQtYjC6.gg5bflnEsOxOinqPgXHzaC
    # Note: Double $$ to escape in compose files
```

## Conclusion

The 5-minute initialization timeout is a security feature, not a bug. The cleanest way to handle it is to use the `--admin-password` or `--admin-password-file` flag at startup, which configures the admin account before anyone can navigate to the UI. For existing installations that have already timed out, removing the data volume and restarting is the quickest recovery path.
