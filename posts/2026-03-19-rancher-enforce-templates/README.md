# How to Enforce Cluster Templates with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Templates, RBAC

Description: Learn how to enforce cluster templates in Rancher so that all new clusters must be created from approved templates.

Enforcing cluster templates in Rancher ensures that every new Kubernetes cluster in your organization meets defined standards. When template enforcement is enabled, users cannot create clusters with ad-hoc configurations. This guide shows you how to set up and manage template enforcement effectively.

## Prerequisites

- Rancher v2.6 or later with admin access
- At least one cluster template already created and configured
- Users and roles configured in Rancher
- An understanding of Rancher RBAC

## Why Enforce Cluster Templates

Without enforcement, any user with cluster creation permissions can provision clusters with arbitrary configurations. This leads to configuration drift, security gaps, and operational inconsistencies. Enforcement ensures that every cluster aligns with your organization's standards for networking, security, and resource management.

## Step 1: Enable Template Enforcement

Enable the enforcement setting at the global level:

1. Log in to Rancher as an administrator.
2. Click the hamburger menu and go to **Global Settings**.
3. Find the setting `cluster-template-enforcement`.
4. Click **Edit** and set the value to `true`.

Alternatively, use the Rancher API:

```bash
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "true"}' \
  "https://rancher.example.com/v3/settings/cluster-template-enforcement"
```

## Step 2: Verify Enforcement Is Active

Confirm the setting is applied:

```bash
curl -s -k \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/settings/cluster-template-enforcement" | jq '.value'
```

The output should return `"true"`.

## Step 3: Prepare Templates for Enforcement

Before enforcing templates, make sure your templates are production-ready:

1. Navigate to **Cluster Management** then **RKE Templates**.
2. Review each template to ensure all critical settings are locked.
3. Verify that templates cover all the infrastructure providers your teams use.
4. Set a default revision for each template.

Check that the following settings are locked in your templates:

```yaml
# Critical locked settings checklist

network:
  plugin: canal  # Locked

services:
  kube-api:
    authorization:
      mode: rbac  # Locked
    audit_log:
      enabled: true  # Locked
    secrets_encryption_config:
      enabled: true  # Locked
  etcd:
    backup_config:
      enabled: true  # Locked
      interval_hours: 6  # Locked
```

## Step 4: Assign Template Access

Make sure all users who need to create clusters have access to at least one template:

1. Open a cluster template.
2. Click **Add Member**.
3. Add users or groups that need access.
4. Set the role to **Member** for users who should use but not modify the template.

```bash
# Add a member to a template via API
curl -s -k \
  -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "accessType": "member",
    "clusterTemplateId": "cattle-global-data:ct-xxxxx",
    "userPrincipalId": "local://user-xxxxx"
  }' \
  "https://rancher.example.com/v3/clusterTemplateRevisions"
```

## Step 5: Test Enforcement

Verify that enforcement works correctly:

1. Log in as a non-admin user who has cluster creation permissions.
2. Navigate to **Cluster Management** and click **Create**.
3. You should see that the **Cluster Template** field is now required.
4. Attempting to create a cluster without selecting a template should be blocked.

Test with the API as well:

```bash
# This should fail when enforcement is enabled
curl -s -k \
  -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "test-cluster",
    "rancherKubernetesEngineConfig": {
      "network": {
        "plugin": "flannel"
      }
    }
  }' \
  "https://rancher.example.com/v3/clusters"
```

The response should indicate that a cluster template is required.

## Step 6: Handle Exceptions

In some cases, you may need to allow specific users to bypass enforcement:

1. Go to **Users & Authentication**.
2. Select the user or group that needs an exception.
3. Assign the **Cluster Template Owner** role at the global level.

Users with admin or cluster-template-owner roles can still create clusters without templates. Limit these roles to platform administrators only.

## Step 7: Monitor Compliance

Set up monitoring to track template compliance across your clusters:

```bash
# List all clusters and their template status
curl -s -k \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/clusters" | \
  jq '.data[] | {
    name: .name,
    template: .clusterTemplateId,
    revision: .clusterTemplateRevisionId,
    state: .state
  }'
```

Create a script to run periodically and alert on non-compliant clusters:

```bash
#!/bin/bash
# check-template-compliance.sh

RANCHER_URL="https://rancher.example.com"
TOKEN="$RANCHER_TOKEN"

clusters=$(curl -s -k \
  -H "Authorization: Bearer $TOKEN" \
  "$RANCHER_URL/v3/clusters" | jq -r '.data[] | select(.clusterTemplateId == null) | .name')

if [ -n "$clusters" ]; then
  echo "WARNING: The following clusters are not using templates:"
  echo "$clusters"
  # Send alert via your preferred notification system
fi
```

## Step 8: Update Enforcement Policies

As your organization grows, refine your enforcement approach:

1. **Create provider-specific templates**: Have separate templates for AWS, Azure, GCP, and on-premises deployments.
2. **Establish a template review process**: Require peer review before template changes are published.
3. **Version template policies**: Track which templates are approved for which environments.

```yaml
# Example: Template naming convention
templates:
  - name: prod-aws-standard
    provider: amazonec2
    environment: production
    locked_fields: [network, rbac, etcd, audit]

  - name: dev-aws-flexible
    provider: amazonec2
    environment: development
    locked_fields: [network, rbac]

  - name: prod-vsphere-standard
    provider: vsphere
    environment: production
    locked_fields: [network, rbac, etcd, audit]
```

## Step 9: Handle Template Enforcement with Terraform

If you use Terraform for Rancher management, configure enforcement in your Terraform code:

```hcl
resource "rancher2_setting" "cluster_template_enforcement" {
  name  = "cluster-template-enforcement"
  value = "true"
}

resource "rancher2_cluster_template" "production" {
  name        = "production-standard"
  description = "Standard production cluster template"

  members {
    access_type       = "member"
    group_principal_id = "local://g-xxxxx"
  }

  template_revisions {
    name    = "v1.0"
    default = true

    cluster_config {
      rke_config {
        network {
          plugin = "canal"
        }
        services {
          kube_api {
            audit_log {
              enabled = true
            }
          }
          etcd {
            backup_config {
              enabled        = true
              interval_hours = 6
              retention      = 30
            }
          }
        }
      }
    }
  }
}
```

## Best Practices

- **Communicate changes early**: Notify teams before enabling enforcement so they understand the new requirements.
- **Provide sufficient templates**: Ensure there are templates for every valid use case to avoid blocking legitimate work.
- **Audit regularly**: Review template assignments and compliance reports on a regular schedule.
- **Maintain escape hatches**: Have a documented process for emergency exceptions that requires management approval.
- **Automate compliance checks**: Integrate template compliance into your CI/CD pipelines and monitoring dashboards.

## Conclusion

Enforcing cluster templates in Rancher is a critical governance step for organizations running multiple Kubernetes clusters. By enabling enforcement, preparing comprehensive templates, and monitoring compliance, you create a secure and consistent foundation for your container infrastructure. The key is balancing control with flexibility so teams can remain productive within defined guardrails.
