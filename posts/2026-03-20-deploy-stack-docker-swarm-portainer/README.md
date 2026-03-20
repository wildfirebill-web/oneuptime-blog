# How to Deploy a Stack on Docker Swarm via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker Swarm, Portainer, Docker Compose, Stack Deployment, DevOps

Description: Learn how to deploy a multi-service Docker stack on a Swarm cluster using Portainer's web interface.

## What Is a Docker Stack?

A Docker Stack is a group of interrelated services that share dependencies and can be orchestrated and scaled together. In Swarm mode, stacks are defined using Compose v3 files and deployed with `docker stack deploy`.

Portainer gives you a visual interface to manage stacks without touching the CLI.

## Prerequisites

- Portainer connected to a Docker Swarm endpoint
- A Compose v3 file for your application

## Deploying a Stack via the Portainer UI

1. In the Portainer sidebar, go to **Stacks** under your Swarm environment.
2. Click **Add stack**.
3. Give the stack a name (lowercase, no spaces).
4. Paste your Docker Compose content into the Web editor, or link to a Git repository.
5. Click **Deploy the stack**.

## Example Stack File

The following Compose v3 file deploys a simple Nginx web server with two replicas across the Swarm:

```yaml
# docker-compose.yml for a Swarm stack deployment
version: "3.8"

services:
  web:
    image: nginx:alpine
    # Deploy with 2 replicas distributed across the swarm
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    ports:
      - "80:80"
    networks:
      - webnet

networks:
  webnet:
    driver: overlay
```

## Using Environment Variables

You can inject environment variables into your stack by using the **Environment variables** section in Portainer before deploying.

```yaml
# Reference environment variables in your Compose file
services:
  app:
    image: myapp:latest
    environment:
      - DB_HOST=${DB_HOST}
      - DB_PASSWORD=${DB_PASSWORD}
```

## Equivalent CLI Deployment

```bash
# Deploy a stack using the CLI
docker stack deploy -c docker-compose.yml my-stack

# List deployed stacks
docker stack ls

# List services in a stack
docker stack services my-stack
```

## Updating a Stack

To update an existing stack in Portainer, navigate to **Stacks**, click the stack name, edit the Compose content, and click **Update the stack**. Portainer handles the rolling update automatically.

## Conclusion

Portainer simplifies Swarm stack deployments with a visual editor, environment variable injection, and one-click updates — ideal for teams who prefer UI-driven workflows over CLI management.
