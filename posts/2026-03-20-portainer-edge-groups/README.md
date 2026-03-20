# How to Create Edge Groups in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Edge Computing, IoT, DevOps

Description: Learn how to create and manage Edge Groups in Portainer to organize remote edge devices and deploy applications to them at scale.

## Introduction

Edge Groups in Portainer are a powerful way to organize and manage your remote edge devices. Whether you're running hundreds of industrial IoT sensors or a distributed fleet of retail kiosks, Edge Groups let you categorize devices logically and deploy stacks to all matching devices simultaneously.

This guide walks you through creating Edge Groups, assigning edge endpoints to them, and using groups for targeted deployments.

## Prerequisites

- Portainer Business Edition (BE) or Community Edition with Edge support
- At least one Portainer Edge Agent deployed and connected
- Admin access to Portainer

## Understanding Edge Groups

Portainer supports two types of Edge Groups:

1. **Static Groups** — You manually assign specific edge endpoints to the group.
2. **Dynamic Groups** — Portainer automatically assigns endpoints to the group based on matching tags.

Dynamic groups are especially useful when you have a large fleet where new devices are frequently added.

## Step 1: Navigate to Edge Groups

1. Log in to your Portainer instance.
2. From the left sidebar, click **Edge Compute**.
3. Select **Edge Groups**.
4. Click **Add Edge Group**.

## Step 2: Configure a Static Edge Group

```yaml
# Example: Tagging edge agents at deployment time using docker-compose
# Use this when deploying the Portainer Edge Agent to your devices

version: "3.8"
services:
  portainer_edge_agent:
    image: portainer/agent:latest
    environment:
      # The unique key provided by Portainer when creating an edge endpoint
      - EDGE_KEY=your_edge_key_here
      # Tags help with dynamic group membership
      - EDGE_TAGS=region=us-west,env=production
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    restart: always
```

For a static group:
1. Enter a **Group Name** (e.g., `US-West-Production`).
2. Under **Group type**, select **Static**.
3. From the **Associated endpoints** list, select the edge endpoints you want to include.
4. Click **Create edge group**.

## Step 3: Configure a Dynamic Edge Group

Dynamic groups use tags to automatically include endpoints:

1. Enter a **Group Name** (e.g., `All-Production-Devices`).
2. Under **Group type**, select **Dynamic**.
3. Add tags that endpoints must match (e.g., `env=production`).
4. Portainer will automatically add any edge endpoint that carries those tags.

```bash
# When deploying the edge agent manually via Docker CLI,
# pass tags as environment variables
docker run -d \
  --name portainer_edge_agent \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  -e EDGE_KEY="your_edge_key_here" \
  -e EDGE_TAGS="region=eu-central,env=staging" \
  portainer/agent:latest
```

## Step 4: Verify Group Membership

After creating a group:
1. Navigate to **Edge Compute > Edge Groups**.
2. Click on the group name.
3. Review the list of associated endpoints.
4. For dynamic groups, verify that all expected tagged endpoints are listed.

## Step 5: Use Edge Groups in Deployments

Edge Groups are used when deploying **Edge Stacks**:

1. Go to **Edge Compute > Edge Stacks**.
2. Click **Add edge stack**.
3. In the **Edge Groups** field, select the group you just created.
4. The stack will be deployed to every endpoint in that group.

## Best Practices

- **Name groups descriptively**: Use names like `Factory-Floor-EU`, `Retail-POS-US-East`, or `Lab-Devices-QA`.
- **Use dynamic groups for large fleets**: Avoids manual assignment as devices scale.
- **Combine tags**: Use multiple tags (e.g., `site=berlin`, `role=gateway`) for fine-grained targeting.
- **Review group membership regularly**: Especially for dynamic groups, ensure no unexpected devices are included.

## Conclusion

Edge Groups in Portainer provide a flexible and scalable mechanism for organizing remote devices. By using static groups for controlled, explicit assignments and dynamic groups for tag-based automation, you can manage even the largest edge fleets with minimal overhead. Once groups are set up, deploying consistent workloads across all matching devices becomes a single-click operation.
