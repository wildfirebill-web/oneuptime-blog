# How to Create Cluster Templates in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Templates, RBAC

Description: Learn how to create and manage cluster templates in Rancher for standardized Kubernetes cluster provisioning across your organization.

Cluster templates in Rancher allow administrators to define standardized configurations for Kubernetes clusters. By using templates, you ensure consistency across environments and reduce the chance of misconfigurations. This guide walks you through creating, managing, and sharing cluster templates in Rancher.

## Prerequisites

Before you begin, make sure you have the following in place:

- Rancher v2.6 or later installed and accessible
- Admin or cluster-owner level permissions in Rancher
- At least one cloud credential configured if provisioning cloud-based clusters
- A basic understanding of Kubernetes cluster components

## Understanding Cluster Templates

Cluster templates are pre-defined cluster configurations that administrators can create and share with other users. They enforce organizational standards by locking down specific settings while allowing flexibility on others. Templates can define networking plugins, Kubernetes versions, node pool configurations, and security policies.

## Step 1: Access the Cluster Templates Section

Navigate to the cluster templates area in Rancher:

1. Log in to your Rancher dashboard.
2. Click on the hamburger menu in the top-left corner.
3. Select **Cluster Management**.
4. Click on **RKE Templates** in the left sidebar under the **Advanced** section.

## Step 2: Create a New Cluster Template

Click the **Add Template** button to start creating a new template.

Fill in the basic information:

```
Template Name: production-standard
Description: Standard production cluster template with Calico networking and CIS hardening
```

## Step 3: Configure the Kubernetes Version

Select the Kubernetes version for your template:

1. Under **Kubernetes Version**, choose the desired version from the dropdown.
2. For production environments, it is recommended to use a stable, well-tested version rather than the latest release.

```yaml
# Example: Specifying Kubernetes version in RKE config
kubernetes_version: v1.28.x-rancher1-1
```

## Step 4: Configure the Network Provider

Choose and configure the network plugin:

1. Under **Network Provider**, select your preferred CNI plugin (Calico, Canal, Flannel, or Weave).
2. Configure plugin-specific options.

For Calico, a common production configuration looks like this:

```yaml
network:
  plugin: calico
  options:
    calico_cloud_provider: none
  calico_network_provider:
    cloud_provider: none
```

For Canal:

```yaml
network:
  plugin: canal
  options:
    canal_iface: ""
    canal_flannel_backend_type: vxlan
```

## Step 5: Configure Authentication and Authorization

Set up the authorization mode:

1. Under **Authorization**, select the authorization mode.
2. For most production environments, use **RBAC**.

```yaml
services:
  kube-api:
    authorization:
      mode: rbac
```

## Step 6: Define Cluster Services Configuration

Configure etcd, the API server, and other cluster services:

```yaml
services:
  etcd:
    backup_config:
      enabled: true
      interval_hours: 6
      retention: 30
    creation: 12h
    retention: 72h
    snapshot: true
  kube-api:
    audit_log:
      enabled: true
      configuration:
        max_age: 30
        max_backup: 10
        max_size: 100
    event_rate_limit:
      enabled: true
    secrets_encryption_config:
      enabled: true
  kubelet:
    extra_args:
      protect-kernel-defaults: "true"
      streaming-connection-idle-timeout: "1800s"
```

## Step 7: Set Template Revision Controls

Control how the template can be modified:

1. Under **Template Revisions**, decide whether users can modify settings.
2. Mark specific fields as **Required** or **Locked** to enforce standards.

Locked fields cannot be changed by users when they create clusters from this template. Required fields must be filled in but can be customized.

## Step 8: Share the Template

Configure who can use the template:

1. Under **Member Access**, click **Add Member**.
2. Search for users or groups.
3. Assign the appropriate role:
   - **Owner**: Can modify and delete the template
   - **Member**: Can use the template to create clusters

```
Member: devops-team
Role: Member

Member: platform-admin
Role: Owner
```

## Step 9: Create a Cluster from the Template

Once your template is saved, users can create clusters from it:

1. Go to **Cluster Management** and click **Create**.
2. Select the infrastructure provider.
3. Under **Cluster Template**, select your template from the dropdown.
4. Fill in any required fields that were not locked.
5. Configure node pools as needed.
6. Click **Create** to provision the cluster.

## Step 10: Manage Template Revisions

As your infrastructure evolves, you may need to update templates:

1. Navigate back to **RKE Templates**.
2. Click on your template name.
3. Click **Add Revision** to create a new version.
4. Make the necessary changes.
5. Optionally set the new revision as the default.

Existing clusters using previous revisions will not be automatically updated. Cluster owners can choose to upgrade to the new revision at their discretion.

## Using the Rancher API for Template Management

You can also manage cluster templates programmatically using the Rancher API:

```bash
# List all cluster templates
curl -s -k \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/clusterTemplates" | jq '.data[].name'

# Create a cluster template via API
curl -s -k \
  -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production-standard",
    "description": "Standard production cluster template",
    "members": [],
    "enabled": true
  }' \
  "https://rancher.example.com/v3/clusterTemplates"
```

## Best Practices

When working with cluster templates, keep these guidelines in mind:

- **Version your templates**: Use descriptive revision names so teams can track changes over time.
- **Lock critical settings**: Lock networking, RBAC, and security configurations to prevent drift.
- **Test before enforcing**: Create a test cluster from your template before mandating its use across the organization.
- **Document customizable fields**: Clearly communicate which settings users can adjust and which are locked.
- **Review regularly**: Schedule periodic reviews of your templates to incorporate security patches and Kubernetes version updates.

## Conclusion

Cluster templates in Rancher provide a powerful way to standardize Kubernetes deployments across your organization. By defining and enforcing templates, you reduce configuration drift, improve security posture, and simplify the cluster provisioning process for your teams. Start with a small number of well-defined templates and iterate as your infrastructure requirements evolve.
