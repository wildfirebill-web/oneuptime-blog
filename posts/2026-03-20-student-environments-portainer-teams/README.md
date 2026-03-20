# How to Set Up Student Environments with Portainer Teams - Teams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Education, Teams, Docker, Multi-Tenant, Training

Description: Use Portainer's Teams feature to create isolated student environments on a shared Docker host, giving each learner their own scoped access to containers, stacks, and volumes.

---

Portainer Teams allow you to segment access to a Docker environment by group. In an educational context, each student or project group gets their own team with scoped permissions - they can deploy and manage containers within their scope without interfering with others. This is the foundation of a practical multi-student Docker lab.

## Step 1: Create a Dedicated Learning Environment

In Portainer, add the shared Docker host as a standalone environment:

1. Go to **Environments > Add Environment**
2. Choose **Docker Standalone**
3. Name it `docker-lab`
4. Connect via the agent or Docker socket

## Step 2: Create Teams

Go to **Settings > Teams > Add Team**:

```text
Team: cohort-2026-a
Team: cohort-2026-b
Team: project-team-1
```

Create one team per class section or project group.

## Step 3: Create Student User Accounts

Go to **Settings > Users > Add User**:

```text
Username: student01
Password: (set or generate)
Role: Standard User (not Administrator)
```

Assign each student to their team by opening the team and clicking **Add member**.

## Step 4: Assign Team Access to the Environment

Open the `docker-lab` environment settings and under **Access control**, assign teams:

```text
cohort-2026-a → Read-Write
cohort-2026-b → Read-Write
```

This ensures students from one cohort cannot see containers or stacks belonging to another cohort.

## Step 5: Pre-deploy Exercise Stacks Per Team

Optionally, pre-deploy starter stacks for each team via Portainer's API so students can hit the ground running:

```bash
# Portainer API - deploy a stack for a specific team

curl -X POST https://portainer:9443/api/stacks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "student01-starter",
    "StackFileContent": "version: \"3.8\"\nservices:\n  web:\n    image: nginx:alpine\n    ports:\n      - \"8100:80\"",
    "Env": []
  }'
```

## Step 6: Configure Resource Limits

In Portainer Business Edition, set resource limits per team to prevent any student from monopolizing the shared host:

- Maximum containers: 5
- Memory limit per container: 512 MB
- CPU shares limit: 1.0

For CE, enforce limits through Docker daemon configuration or at the container level in the deployment form.

## Step 7: Student Workflow

Once configured, students log in to the Portainer UI and see only their team's resources:

1. Deploy containers from **App Templates** or the container creation form
2. View logs and use the Console for debugging
3. Create stacks for multi-service exercises
4. Browse volumes and inspect network configurations

## Summary

Portainer Teams provide the multi-tenancy needed for a shared Docker learning lab. Each student works in an isolated scope, instructors retain admin visibility across all teams, and shared resources are partitioned fairly. The entire setup requires only a single Docker host and Portainer CE - no Kubernetes, no cloud infrastructure required.
