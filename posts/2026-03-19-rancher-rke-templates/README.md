# How to Use RKE Templates for Consistent Cluster Provisioning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Templates, RKE

Description: A step-by-step guide to using RKE templates in Rancher for consistent and repeatable Kubernetes cluster provisioning.

RKE (Rancher Kubernetes Engine) templates allow you to define reusable cluster configurations that enforce organizational standards while still giving teams flexibility where needed. This guide covers how to create, customize, and deploy clusters using RKE templates in Rancher.

## Prerequisites

- Rancher v2.6 or later
- Admin or standard user permissions with template creation rights
- Familiarity with RKE cluster configuration options
- At least one infrastructure provider configured in Rancher

## What Are RKE Templates

RKE templates are versioned cluster configuration blueprints. Each template can have multiple revisions, and administrators can control which settings are locked (immutable) and which are open for customization. This separation enables platform teams to enforce security and networking standards while allowing development teams to adjust resource-specific settings.

## Step 1: Plan Your Template Structure

Before creating a template, plan which settings to standardize:

| Setting | Locked | Reason |
|---------|--------|--------|
| Kubernetes Version | No | Allow teams to test new versions |
| Network Plugin | Yes | Ensure consistent networking |
| RBAC | Yes | Security requirement |
| etcd Backup | Yes | Data protection |
| Audit Logging | Yes | Compliance requirement |
| Pod Security Policies | No | Varies by workload |

## Step 2: Create the RKE Template

Navigate to **Cluster Management** then **RKE Templates** and click **Add Template**.

Enter the template details:

```plaintext
Name: standard-rke-cluster
Description: Organization-standard RKE cluster configuration

```

## Step 3: Configure RKE Options

Set the core RKE configuration for your template:

```yaml
# RKE Configuration

kubernetes_version: v1.28.x-rancher1-1

# Network configuration
network:
  plugin: canal
  options:
    canal_flannel_backend_type: vxlan

# Authentication
services:
  kube-api:
    authorization:
      mode: rbac
    service_cluster_ip_range: 10.43.0.0/16
    extra_args:
      anonymous-auth: "false"
      profiling: "false"
  kube-controller:
    cluster_cidr: 10.42.0.0/16
    service_cluster_ip_range: 10.43.0.0/16
    extra_args:
      terminated-pod-gc-threshold: "1000"
      profiling: "false"
  kubelet:
    cluster_domain: cluster.local
    cluster_dns_server: 10.43.0.10
    extra_args:
      max-pods: "110"
      serialize-image-pulls: "false"
  scheduler:
    extra_args:
      profiling: "false"
```

## Step 4: Configure etcd Settings

Set up etcd backup and recovery options:

```yaml
services:
  etcd:
    backup_config:
      enabled: true
      interval_hours: 6
      retention: 30
      s3_backup_config:
        access_key: ""
        bucket_name: "etcd-backups"
        endpoint: "s3.amazonaws.com"
        folder: ""
        region: "us-east-1"
        secret_key: ""
    creation: 12h
    extra_args:
      election-timeout: "5000"
      heartbeat-interval: "500"
    retention: 72h
    snapshot: true
```

## Step 5: Enable Security Features

Add security-related configurations:

```yaml
services:
  kube-api:
    audit_log:
      enabled: true
      configuration:
        max_age: 30
        max_backup: 10
        max_size: 100
        path: /var/log/kube-audit/audit-log.json
        format: json
        policy:
          apiVersion: audit.k8s.io/v1
          kind: Policy
          rules:
            - level: Metadata
              resources:
                - group: ""
                  resources: ["secrets", "configmaps"]
            - level: RequestResponse
              resources:
                - group: ""
                  resources: ["pods"]
    secrets_encryption_config:
      enabled: true
    event_rate_limit:
      enabled: true
```

## Step 6: Lock Template Fields

After configuring your template, lock the fields that should not be modified:

1. Next to each configuration section, look for the lock icon.
2. Click the lock to toggle between **Locked** and **Unlocked**.
3. Lock the following sections at minimum:
   - Network Provider
   - RBAC configuration
   - etcd backup settings
   - Audit log configuration
   - Secrets encryption

## Step 7: Create Template Revisions

Save the initial template, then create revisions as your standards evolve:

1. Open the template.
2. Click **Add Revision**.
3. Update the Kubernetes version or other settings.
4. Give the revision a descriptive name like `v1.1-k8s-1.29`.
5. Optionally set it as the default revision.

```bash
# You can also manage revisions via API
curl -s -k \
  -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "clusterTemplateId": "cattle-global-data:ct-xxxxx",
    "clusterConfig": {
      "rancherKubernetesEngineConfig": {
        "kubernetesVersion": "v1.29.x-rancher1-1"
      }
    },
    "name": "v1.1-k8s-1.29",
    "enabled": true
  }' \
  "https://rancher.example.com/v3/clusterTemplateRevisions"
```

## Step 8: Provision a Cluster from the Template

Create a new cluster using the RKE template:

1. Go to **Cluster Management** and click **Create**.
2. Select **Custom** or your preferred infrastructure provider.
3. In the **Cluster Template** dropdown, select your template.
4. Choose the template revision to use.
5. Configure any unlocked fields as needed.
6. Set up your node pools with appropriate roles:
   - **etcd**: At least 3 nodes for high availability
   - **control plane**: At least 2 nodes
   - **worker**: As many as your workloads require

```bash
# Register nodes with specific roles
docker run -d --privileged --restart=unless-stopped \
  --net=host \
  -v /etc/kubernetes:/etc/kubernetes \
  -v /var/run:/var/run \
  rancher/rancher-agent:v2.8.x \
  --server https://rancher.example.com \
  --token <cluster-token> \
  --etcd --controlplane
```

## Step 9: Update Clusters to New Revisions

When you create a new template revision, update existing clusters:

1. Go to the cluster's dashboard.
2. Click the three-dot menu and select **Edit Config**.
3. Under **Cluster Template Revision**, select the new revision.
4. Review the changes that will be applied.
5. Click **Save** to begin the upgrade.

## Step 10: Export and Import Templates

Share templates across Rancher installations:

```bash
# Export a template
curl -s -k \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/clusterTemplates/cattle-global-data:ct-xxxxx" | \
  jq 'del(.id, .uuid, .created, .links, .actions)' > template-export.json

# Import to another Rancher instance
curl -s -k \
  -X POST \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d @template-export.json \
  "https://other-rancher.example.com/v3/clusterTemplates"
```

## Best Practices

- **Start with a minimal locked configuration**: Only lock settings that are truly mandatory. You can always lock more later.
- **Use descriptive revision names**: Include the Kubernetes version and date in revision names for easy tracking.
- **Test revisions in staging**: Always test new template revisions with a staging cluster before rolling them out to production.
- **Automate template management**: Use the Rancher API or Terraform provider to manage templates as code.
- **Monitor template compliance**: Regularly audit clusters to ensure they are using approved template revisions.

## Conclusion

RKE templates in Rancher bring consistency and governance to your Kubernetes cluster provisioning workflow. By thoughtfully designing your templates with the right balance of locked and flexible settings, you empower teams to move quickly while maintaining organizational standards. Combine templates with Rancher's RBAC system and you have a robust platform for multi-team Kubernetes management.
