# How to Deploy a Container from a Template in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Templates, Containers, DevOps

Description: Step-by-step guide to deploying a single container using Portainer's built-in and custom application templates.

## Introduction

Container templates in Portainer provide a fast path to deploying single-container applications without needing to know all the Docker run flags. Templates pre-configure image names, port mappings, environment variables, and volumes, so you only need to fill in the specifics for your environment. This guide walks through deploying a container from a template.

## Prerequisites

- Portainer CE or BE installed
- A Docker standalone environment connected to Portainer
- Basic understanding of Docker container concepts

## What Makes a Container Template

A container template in Portainer defines:

```json
{
  "type": 1,
  "title": "Nginx",
  "description": "High performance web server",
  "categories": ["webserver"],
  "platform": "linux",
  "logo": "https://portainer-io-assets.sfo2.digitaloceanspaces.com/logos/nginx.png",
  "image": "nginx:latest",
  "ports": ["80/tcp", "443/tcp"],
  "volumes": [
    { "container": "/usr/share/nginx/html" },
    { "container": "/etc/nginx" }
  ],
  "restart_policy": "unless-stopped"
}
```

Type `1` is a container template (type `2` is a stack template).

## Step 1: Open App Templates

1. In Portainer, select your Docker environment
2. Click **App Templates** in the left navigation
3. The template catalog loads showing all available templates

## Step 2: Filter to Container Templates

By default, both container and stack templates are shown. To see only container templates:

- Look for the **Type** filter or use the search to find your desired app
- Container templates typically show a single container icon

## Step 3: Select Your Template

Click on the template you want to deploy. A configuration panel expands below the template card.

For this example, we will deploy **MySQL**:

```text
Template: MySQL
Image:    mysql:8
```

## Step 4: Configure the Container

Fill in the configuration fields:

### Basic Settings

```text
Name:     my-mysql          # Unique container name on this host
```

### Port Mappings

```text
3306 → 3306    # Map MySQL port (change host port if needed)
```

### Environment Variables

The template pre-populates common MySQL variables:

```text
MYSQL_ROOT_PASSWORD:  [enter-secure-password]
MYSQL_DATABASE:       mydb
MYSQL_USER:           myuser
MYSQL_PASSWORD:       [enter-secure-password]
```

### Volume Configuration

```text
/var/lib/mysql → /data/mysql    # Persist data to host path
```

Or use a named volume by leaving the host path blank - Portainer creates a named volume automatically.

### Network

```sql
Network: bridge    # Default; or select a custom network
```

### Restart Policy

```text
Restart policy: Unless stopped    # Restart unless manually stopped
```

## Step 5: Advanced Options (Optional)

Expand the **Advanced options** section for:

- **Labels** - Add Docker labels for organization or proxy routing
- **CPU limit** - Constrain CPU usage
- **Memory limit** - Set memory limits (e.g., 512m)
- **Privileged mode** - Run with extended privileges (use with caution)

```text
Memory limit: 512m
CPU limit:    0.5
```

## Step 6: Deploy the Container

1. Review all settings
2. Click **Deploy the container**
3. Portainer pulls the image (if not already cached) and creates the container
4. You are redirected to the **Containers** list

## Step 7: Verify the Deployment

1. Find your container in the **Containers** list (it should show **Running**)
2. Click the container name to view details
3. Check the **Logs** tab for startup messages:

```text
2024-01-15T10:00:00 [Note] MySQL init process done. Ready for start up.
2024-01-15T10:00:01 [Note] mysqld: ready for connections.
```

4. Test connectivity:

```bash
# Connect to MySQL from another container

docker exec -it my-mysql mysql -u root -p

# Or test the port from the host
mysql -h 127.0.0.1 -P 3306 -u root -p
```

## Modifying After Deployment

Container templates create a standard Docker container. You can:

- Restart via the **Containers** list (click the play/stop icons)
- Edit environment variables via **Container details > Duplicate/Edit**
- Update the image by pulling a new version and recreating

## Creating Your Own Container Template

You can save any container configuration as a custom template for reuse:

1. Go to **App Templates > Custom Templates**
2. Click **+ Add custom template**
3. Choose **Container** type
4. Fill in the template JSON definition

## Conclusion

Deploying containers from templates in Portainer simplifies the process of launching well-known applications. Templates handle the complexity of Docker configuration, letting you focus on the values specific to your environment. Once comfortable with the built-in templates, consider building custom templates to standardize deployments across your team.
