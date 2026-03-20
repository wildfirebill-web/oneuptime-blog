# How to Set Up Portainer with a Custom Admin Password on First Launch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Security, Automation, DevOps

Description: Learn how to pre-configure a Portainer admin password at launch time using the --admin-password flag for automated deployments.

---

By default, Portainer presents a web-based setup wizard on first launch where you create the admin account. For automated or scripted deployments, you can pre-set the admin password using a startup flag, bypassing the interactive setup screen.

## Option 1: Pass a Hashed Password at Runtime

Portainer accepts a bcrypt-hashed password via the `--admin-password` flag. First, generate the hash:

```bash
# Install htpasswd to generate a bcrypt hash

# On Ubuntu/Debian:
sudo apt-get install apache2-utils

# Generate bcrypt hash (cost factor 12 recommended)
# Replace 'yourpassword' with your actual password
htpasswd -nbB admin "yourpassword" | cut -d: -f2
# Output example: $2y$12$abc123...
```

## Option 2: Use the admin-password-file Flag

For more secure automation, store the hash in a file rather than passing it on the command line:

```bash
# Create the hash and save to a file
htpasswd -nbB admin "yourpassword" | cut -d: -f2 > /tmp/portainer_admin_hash.txt

# Mount the file into the container and reference it
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  -v /tmp/portainer_admin_hash.txt:/tmp/portainer_password \
  portainer/portainer-ce:latest \
  --admin-password-file /tmp/portainer_password
```

## Option 3: Inline Hash in docker run

Pass the hash directly on the command line (less secure, appears in shell history):

```bash
# Generate hash inline and pass to Portainer
HASH=$(htpasswd -nbB admin "yourpassword" | cut -d: -f2)

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --admin-password "$HASH"
```

## Docker Compose Example

For infrastructure-as-code deployments, use Docker Compose with an environment variable:

```yaml
# docker-compose.yml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - "8000:8000"
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
      # Mount the pre-generated hash file
      - ./portainer_admin_hash.txt:/tmp/portainer_password:ro
    command: --admin-password-file /tmp/portainer_password

volumes:
  portainer_data:
```

## Verify the Setup

After starting with a pre-configured password, the setup wizard is skipped. Log in directly at `https://localhost:9443` using `admin` and the password you hashed.

## Security Best Practices

- Never pass plain text passwords on the command line
- Use Docker secrets for production deployments on Swarm
- Rotate the admin password after initial setup via **Settings > Users**
- Minimum password length is 12 characters in Portainer

---

*Automate your infrastructure monitoring with [OneUptime](https://oneuptime.com) alongside your Portainer deployments.*
