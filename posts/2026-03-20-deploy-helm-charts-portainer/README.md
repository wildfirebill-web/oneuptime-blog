# How to Deploy Helm Charts in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Helm, Application Deployment, DevOps

Description: Learn how to browse, configure, and deploy Helm charts to your Kubernetes cluster directly from the Portainer UI.

## What Is Helm?

Helm is the package manager for Kubernetes. Helm charts are pre-packaged Kubernetes application templates with configurable values. Portainer integrates with Helm, letting you deploy charts without using the CLI.

## Accessing Helm in Portainer

1. Select your Kubernetes environment in Portainer.
2. Go to **Applications > Helm charts** (or **Helm** in the sidebar).
3. Portainer shows charts from configured Helm repositories.

## Adding a Helm Repository

Before deploying charts, add a repository:

1. Go to **Settings > Helm** (or from the environment settings).
2. Click **Add Helm repository**.
3. Enter the repository URL (e.g., `https://charts.bitnami.com/bitnami`).
4. Click **Save**.

Popular repositories:
```
https://charts.bitnami.com/bitnami
https://helm.nginx.com/stable
https://prometheus-community.github.io/helm-charts
https://grafana.github.io/helm-charts
```

## Deploying a Chart via the UI

1. Browse the chart catalog and click on a chart.
2. Review the chart description and available versions.
3. Click **Install**.
4. Configure:
   - **Release name**: A unique name for this deployment.
   - **Namespace**: Target Kubernetes namespace.
   - **Chart version**: Select a specific version.
   - **Values**: Customize the chart's `values.yaml` via the inline editor.
5. Click **Install** to deploy.

## Customizing Values

Portainer shows the default `values.yaml` and lets you override specific settings:

```yaml
# Example: Customizing nginx chart values
replicaCount: 2

service:
  type: LoadBalancer
  port: 80

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Enable autoscaling
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
```

## Equivalent CLI Commands

```bash
# Add a Helm repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search for available charts
helm search repo bitnami/nginx

# Install a chart with custom values
helm install my-nginx bitnami/nginx \
  --namespace production \
  --create-namespace \
  --set replicaCount=2 \
  --set service.type=LoadBalancer

# Install with a values file
helm install my-nginx bitnami/nginx \
  --namespace production \
  -f custom-values.yaml
```

## Upgrading a Deployed Chart

In Portainer, navigate to the installed release and click **Upgrade**. You can change the chart version and update values without downtime.

## Conclusion

Portainer's Helm integration provides a visual chart browser, values editor, and upgrade manager. It makes Helm deployments accessible to developers who aren't fluent with the Helm CLI while maintaining full customization capability.
