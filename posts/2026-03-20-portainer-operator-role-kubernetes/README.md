# How to Use the Operator Role in Portainer for Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Operator, RBAC, Access Control

Description: Configure and use the Operator role in Portainer for Kubernetes environments to allow application deployment without infrastructure management access.

## Introduction

The Operator role in Portainer is designed specifically for Kubernetes environments. It provides a middle ground between the Standard User and Administrator roles — operators can deploy and manage applications but cannot modify cluster-level resources or infrastructure settings. This guide covers the Operator role's capabilities and configuration.

## Operator vs. Standard User vs. Administrator in Kubernetes

| Capability | Helpdesk | Standard User | Operator | Administrator |
|-----------|---------|--------------|---------|--------------|
| View workloads | ✓ | ✓ | ✓ | ✓ |
| Scale deployments | ✗ | ✓ | ✓ | ✓ |
| Deploy applications | ✗ | ✓ | ✓ | ✓ |
| Restart pods | ✗ | ✓ | ✓ | ✓ |
| Delete namespaces | ✗ | ✗ | ✗ | ✓ |
| Create namespaces | ✗ | ✗ | ✗ | ✓ |
| Manage storage classes | ✗ | ✗ | ✗ | ✓ |
| Access cluster nodes | ✗ | ✗ | ✗ | ✓ |

## What Operators Can Do

In assigned Kubernetes environments, operators can:

- **View**: All workloads, services, configmaps, secrets (subject to namespace restrictions)
- **Deploy**: Create and update deployments, DaemonSets, StatefulSets, Jobs
- **Manage**: Scale replicas, update images, restart deployments
- **Troubleshoot**: Access pod logs, exec into containers
- **Services**: Create/update services and ingresses in their namespaces

## What Operators Cannot Do

- Create or delete namespaces
- Modify ClusterRoles or ClusterRoleBindings
- Access node-level information
- Modify storage classes or persistent volumes (cluster-level)
- Access environments not assigned to them

## Assigning the Operator Role to a Team

```bash
TOKEN=$(curl -s -X POST \
  https://portainer.example.com/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"adminpassword"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Assign team 3 (developers) to Kubernetes environment 5 with Operator role
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://portainer.example.com/api/endpoints/5/teamaccesspolicies \
  -d '{
    "3": {"RoleId": 2}
  }'
# RoleId 2 = Operator role
```

## Namespace Access for Operators

Combine the Operator role with namespace-level access control for fine-grained Kubernetes access:

### Step 1: Enable Namespace Access Control

In the Kubernetes environment settings:
1. Go to the environment → **Settings**
2. Enable **Namespace access management**

### Step 2: Assign Namespaces to Teams

```bash
# Assign team to a specific namespace
# (This is done via the Kubernetes environment UI in Portainer)
# Navigate to: Environments → [K8s Env] → Namespaces → [Namespace] → Access

# Or via API for the Kubernetes environment
ENDPOINT_ID=5
NAMESPACE="development"
TEAM_ID=3

curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "https://portainer.example.com/api/endpoints/${ENDPOINT_ID}/kubernetes/namespaces/${NAMESPACE}/access" \
  -d "{
    \"teamAccessPolicies\": {
      \"${TEAM_ID}\": {
        \"RoleId\": 2
      }
    }
  }"
```

## Typical Kubernetes Team Structure

```
Organization Kubernetes Access Design:

Platform Team → Administrator role → All environments
DevOps Team   → Operator role     → Production environment
Dev Team      → Operator role     → Development + Staging environments
QA Team       → Standard User     → Testing environment
Support Team  → Helpdesk role     → All environments (read-only)
```

## Using the Operator Role in Practice

As an Operator, when deploying a new application:

1. Navigate to the assigned Kubernetes environment
2. Go to **Applications** → **Add application**
3. Create the deployment with the required configuration
4. The application is deployed to the allowed namespace

Scale a deployment:
1. Navigate to **Applications** → find the deployment
2. Click the deployment name
3. Update the **Replicas** count
4. Click **Update**

## Conclusion

The Operator role is ideal for development teams working in Kubernetes environments where the platform team wants to maintain control over cluster infrastructure while empowering developers to self-service their application deployments. Combined with namespace-level access control, it provides the exact level of access needed — enough to be productive, not enough to cause cluster-level issues.
