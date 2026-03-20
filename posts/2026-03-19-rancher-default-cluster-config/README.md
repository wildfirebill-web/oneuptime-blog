# How to Set Default Cluster Configuration in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Cluster Templates

Description: Learn how to configure default cluster settings in Rancher to streamline cluster provisioning and maintain consistency.

Setting default cluster configurations in Rancher helps ensure that every new cluster starts with your organization's preferred settings. This guide walks you through configuring defaults for Kubernetes versions, networking, storage, and other critical cluster parameters.

## Prerequisites

- Rancher v2.6 or later installed
- Administrator access to Rancher
- Basic understanding of Kubernetes cluster configuration
- At least one infrastructure provider configured

## Understanding Default Configuration

Default cluster configuration in Rancher works at multiple levels. Global defaults apply to all new clusters unless overridden. Template defaults apply when a specific template is selected. Understanding this hierarchy helps you design an effective configuration strategy.

## Step 1: Configure Global Default Kubernetes Version

Set the default Kubernetes version for new clusters:

1. Log in to Rancher as an administrator.
2. Navigate to **Global Settings**.
3. Find `k8s-version-to-images` and `k8s-version`.
4. Set your preferred default version.

```bash
# Set default Kubernetes version via API

curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "v1.28.x-rancher1-1"}' \
  "https://rancher.example.com/v3/settings/k8s-version"
```

## Step 2: Configure Default Network Settings

Set up default networking parameters:

1. Go to **Global Settings**.
2. Locate the network-related settings.
3. Configure the defaults.

```bash
# Set default cluster CIDR
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "10.42.0.0/16"}' \
  "https://rancher.example.com/v3/settings/cluster-cidr"

# Set default service cluster IP range
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "10.43.0.0/16"}' \
  "https://rancher.example.com/v3/settings/service-cluster-ip-range"

# Set default cluster DNS
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "10.43.0.10"}' \
  "https://rancher.example.com/v3/settings/cluster-dns-server"
```

## Step 3: Set Default Pod Security Configuration

Configure default pod security settings:

```yaml
# Default Pod Security Admission configuration
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: PodSecurity
    configuration:
      apiVersion: pod-security.admission.config.k8s.io/v1
      kind: PodSecurityConfiguration
      defaults:
        enforce: "baseline"
        enforce-version: "latest"
        audit: "restricted"
        audit-version: "latest"
        warn: "restricted"
        warn-version: "latest"
```

Apply this through the Rancher UI:

1. Navigate to **Cluster Management**.
2. Click **Create** to see the default configuration form.
3. Under **Security**, configure Pod Security Admission settings.

## Step 4: Configure Default etcd Settings

Set organization-wide etcd backup defaults:

```yaml
services:
  etcd:
    backup_config:
      enabled: true
      interval_hours: 12
      retention: 14
    creation: 12h
    retention: 72h
    snapshot: true
    extra_args:
      quota-backend-bytes: "8589934592"
```

To set these as defaults in the Rancher settings:

```bash
# Configure default etcd backup interval
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "12"}' \
  "https://rancher.example.com/v3/settings/etcd-backup-interval"

# Configure default etcd backup retention
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "14"}' \
  "https://rancher.example.com/v3/settings/etcd-backup-retention"
```

## Step 5: Set Default Monitoring Configuration

Enable monitoring defaults for all new clusters:

1. Navigate to **Apps & Marketplace** in the global scope.
2. Configure the default monitoring values.

Create a default values file for the monitoring chart:

```yaml
# monitoring-defaults.yaml
prometheus:
  prometheusSpec:
    resources:
      requests:
        cpu: 750m
        memory: 750Mi
      limits:
        cpu: 1000m
        memory: 2000Mi
    retention: 7d
    retentionSize: 10GB
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 50Gi

grafana:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi

alertmanager:
  alertmanagerSpec:
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
```

## Step 6: Configure Default Resource Quotas

Set default resource quotas that apply to new projects:

1. Go to **Cluster Management**.
2. Select a cluster.
3. Navigate to **Projects/Namespaces**.
4. Click on the default project.
5. Under **Resource Quotas**, set the defaults.

```yaml
# Default resource quota for new projects
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-project-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
```

## Step 7: Set Default Container Resource Limits

Configure default container resource limits:

```yaml
# LimitRange for default container resources
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "2"
        memory: 4Gi
      min:
        cpu: 50m
        memory: 64Mi
```

Apply this as a default for all new namespaces:

1. Go to the cluster's **Advanced** settings.
2. Set the LimitRange as the default for new namespaces.

## Step 8: Configure Default Logging

Set up default logging configuration:

```bash
# Set default system log level
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "info"}' \
  "https://rancher.example.com/v3/settings/server-log-level"
```

## Step 9: Set Default Agent Configuration

Configure default cluster and node agent settings:

```bash
# Set default agent image
curl -s -k \
  -X PUT \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value": "rancher/rancher-agent:v2.8.x"}' \
  "https://rancher.example.com/v3/settings/agent-image"
```

## Step 10: Document and Distribute Defaults

Create documentation for your default configuration:

```bash
# Export all current settings
curl -s -k \
  -H "Authorization: Bearer $RANCHER_TOKEN" \
  "https://rancher.example.com/v3/settings" | \
  jq '.data[] | select(.value != null and .value != "") | {name: .name, value: .value}' \
  > rancher-defaults.json
```

## Best Practices

- **Review defaults quarterly**: Kubernetes and Rancher evolve rapidly, so review your default settings on a regular cadence.
- **Use templates alongside defaults**: Defaults provide a baseline, while templates enforce specific configurations for different use cases.
- **Test in staging first**: Always verify new default settings in a staging environment before applying them to production.
- **Keep documentation current**: Maintain a living document that explains each default and why it was chosen.
- **Version control your settings**: Store your default configuration in Git to track changes over time.

## Conclusion

Configuring default cluster settings in Rancher creates a strong foundation for consistent Kubernetes deployments. By setting sensible defaults for networking, security, resource management, and monitoring, you reduce the effort required to provision new clusters while maintaining organizational standards. Combine defaults with cluster templates for a comprehensive governance approach.
