# How to Configure Standard User Permissions in Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Standard User, RBAC, Permissions, Access Control

Description: Configure what Standard Users can see and do in Portainer environments including resource visibility and deployment capabilities.

## Introduction

The Standard User role is the default role for most Portainer users - it provides full management capabilities within assigned environments without global admin privileges. This guide covers what Standard Users can do, how to configure their environment access, and how to restrict capabilities further using resource controls.

## Standard User Capabilities

In environments they have access to, Standard Users can:

**Container Management:**
- Create, start, stop, restart, kill, pause containers
- Remove containers
- Access container logs
- Execute commands in running containers (terminal)
- Inspect container configuration

**Image Management:**
- Pull images from registries
- Build images from Dockerfiles
- Tag and remove images

**Stack/Service Management:**
- Deploy and update stacks from compose files
- Update and remove stacks
- Manage services in Docker Swarm

**Volume and Network Management:**
- Create and remove volumes
- Create and remove networks
- Manage network configurations

**Registry Access:**
- Pull from registries the admin has made available

## What Standard Users Cannot Do

- Add, edit, or remove environments
- Create or manage other users
- Configure global settings (authentication, templates)
- Access environments not assigned to them
- Manage registries (only use them)

## Assigning Standard User Access to Environments

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Assign team 2 (developers) to environment 1 with Standard User role

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/teamaccesspolicies \
  -d '{
    "2": {"RoleId": 4}
  }'
# RoleId 4 = Standard User

# Assign individual user (ID: 5) to environment 1
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/1/useraccesspolicies \
  -d '{
    "5": {"RoleId": 4}
  }'
```

Resource Controls (Ownership)

Portainer supports resource-level ownership to control which users can manage specific containers:

### Public Resources
All Standard Users with access to the environment can manage public resources.

### Restricted Resources
Only the owner (and admins) can manage restricted resources.

When a Standard User creates a container, they can set it as:
- **Public**: Visible and manageable by all team members
- **Restricted**: Only the creator and designated users/teams can manage it

```bash
# Create a container with restricted access (via API)
curl -X POST \
  -H "Authorization: Bearer $USER_TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoints/1/docker/containers/create?name=my-private-container" \
  -d '{
    "Image": "nginx:latest",
    "Labels": {
      "io.portainer.accesscontrol.public": "false",
      "io.portainer.accesscontrol.users": "5"
    }
  }'
```

## Configuring Available Docker Features

Admins can control which Docker features are available to Standard Users per environment:

1. Go to **Environments** → select environment → **Settings**
2. Under **Security settings**:
   - **Allow users to use the host PID namespace**: Off by default (security risk)
   - **Allow users to use the host network**: Off by default
   - **Allow users to use privileged containers**: Off by default
   - **Allow users to use volumes mounted on the host**: Off by default

These settings apply to all Standard Users in that environment.

## Available Registries for Standard Users

Admins must explicitly enable registries for each environment:

1. Go to **Environments** → select environment → **Registries**
2. Click **Add registry**
3. Select from globally configured registries

Standard Users can then pull images from enabled registries but cannot configure new ones.

## Conclusion

The Standard User role provides a comprehensive set of container management capabilities while keeping global administration centralized. Fine-tune access using environment-level security settings to enable or disable specific Docker capabilities, and use resource controls to allow ownership-based access to individual containers and stacks. Most development and operations staff should be Standard Users in their assigned environments.
