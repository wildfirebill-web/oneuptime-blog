# How to Set Up a Shared Services Project in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Project, Namespace, Multi-Tenancy, Network Isolation

Description: Learn how to create and configure a shared services project in Rancher for hosting cross-cutting infrastructure like monitoring, logging, and ingress.

In a multi-tenant Rancher environment, some services need to be accessible by all projects: monitoring, logging, ingress controllers, certificate management, and service meshes. A shared services project provides a centralized home for these cross-cutting concerns while maintaining proper access controls. This guide shows how to set up and manage a shared services project.

## Prerequisites

- Rancher v2.7+ with cluster owner or administrator access
- Multiple tenant projects in a cluster
- Understanding of which services need to be shared
- A CNI that supports NetworkPolicy

## Why a Shared Services Project

Without a shared services project, you face two bad options:

1. **Deploy shared tools in each project**: Wastes resources and creates maintenance burden.
2. **Deploy in the System project**: Mixes your applications with Kubernetes system components.

A dedicated shared services project keeps cross-cutting infrastructure organized, properly isolated, and independently manageable.

## Step 1: Create the Shared Services Project

**Via the Rancher UI:**

1. Navigate to your cluster.
2. Go to **Cluster > Projects/Namespaces**.
3. Click **Create Project**.
4. Set the name to `shared-services`.
5. Set the description to `Cross-cutting infrastructure services accessible by all projects`.
6. Configure resource quotas appropriate for shared infrastructure.
7. Click **Create**.

**Via kubectl:**

```yaml
apiVersion: management.cattle.io/v3
kind: Project
metadata:
  generateName: p-
  namespace: c-m-xxxxx
spec:
  displayName: shared-services
  description: "Cross-cutting infrastructure services"
  clusterName: c-m-xxxxx
  resourceQuota:
    limit:
      pods: "200"
      requestsCpu: "16000m"
      requestsMemory: "64Gi"
      limitsCpu: "32000m"
      limitsMemory: "128Gi"
      persistentVolumeClaims: "50"
    usedLimit: {}
  namespaceDefaultResourceQuota:
    limit:
      pods: "50"
      requestsCpu: "4000m"
      requestsMemory: "16Gi"
```

```bash
kubectl apply -f shared-services-project.yaml
```

## Step 2: Create Namespaces for Shared Services

Organize shared services into dedicated namespaces:

```bash
# Create namespaces

for ns in monitoring logging ingress cert-manager shared-databases; do
  cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: $ns
  annotations:
    field.cattle.io/projectId: "c-m-xxxxx:p-shared"
  labels:
    field.cattle.io/projectId: "p-shared"
    project: shared-services
    purpose: infrastructure
EOF
done
```

## Step 3: Configure RBAC for the Shared Services Project

The platform team owns the shared services project. Tenant teams get read-only access:

```hcl
# Platform team - full management
resource "rancher2_project_role_template_binding" "shared_platform_owner" {
  name               = "shared-platform-owner"
  project_id         = rancher2_project.shared_services.id
  role_template_id   = "project-owner"
  group_principal_id = data.rancher2_principal.platform_team.id
}

# All users - read-only access to monitoring dashboards
resource "rancher2_project_role_template_binding" "shared_all_readonly" {
  name               = "shared-all-readonly"
  project_id         = rancher2_project.shared_services.id
  role_template_id   = "read-only"
  group_principal_id = data.rancher2_principal.all_developers.id
}
```

## Step 4: Configure Network Policies for Shared Services

The shared services project needs a different network policy model than tenant projects. Shared services must accept traffic from all tenant projects:

**Allow all projects to reach shared services:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
  namespace: monitoring
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Allow from all namespaces (all tenants)
    - from:
        - namespaceSelector: {}
```

**Allow shared services to scrape metrics from all projects:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: monitoring
spec:
  podSelector:
    matchLabels:
      app: prometheus
  policyTypes:
    - Egress
  egress:
    # Allow Prometheus to reach all namespaces for scraping
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 9090
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 9100
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

**Allow tenant namespaces to reach the ingress controller:**

Apply this policy in each tenant namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: team-a-production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              project: shared-services
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
```

## Step 5: Deploy Monitoring in the Shared Project

Install Prometheus and Grafana in the monitoring namespace:

```bash
# Using Rancher's monitoring integration
# Navigate to Cluster > Cluster Tools > Monitoring
# Or install via Helm:

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=<secure-password>
```

## Step 6: Deploy Logging in the Shared Project

Set up centralized logging:

```bash
# Install Loki for log aggregation
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace logging \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi
```

## Step 7: Deploy the Ingress Controller

Install a shared ingress controller:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress \
  --set controller.replicaCount=3 \
  --set controller.resources.requests.cpu=250m \
  --set controller.resources.requests.memory=256Mi
```

Label the ingress namespace for NetworkPolicy selectors:

```bash
kubectl label namespace ingress app=ingress-controller
```

## Step 8: Deploy Certificate Management

Install cert-manager for automated TLS:

```bash
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set installCRDs=true \
  --set resources.requests.cpu=50m \
  --set resources.requests.memory=64Mi
```

Create a ClusterIssuer that all projects can use:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@example.com
    privateKeySecretRef:
      name: letsencrypt-production-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

## Step 9: Set Up Cross-Project Service Access

Allow tenant services to connect to shared databases or caches:

```yaml
# Shared Redis in the shared-databases namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-redis-access
  namespace: shared-databases
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
    - Ingress
  ingress:
    - from:
        # Allow specific tenant projects
        - namespaceSelector:
            matchExpressions:
              - key: project
                operator: In
                values:
                  - team-a
                  - team-b
                  - team-c
      ports:
        - protocol: TCP
          port: 6379
```

Create a Kubernetes Service that tenant namespaces can reference:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shared-redis
  namespace: shared-databases
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
```

Tenant applications connect using: `shared-redis.shared-databases.svc.cluster.local:6379`

## Step 10: Document and Communicate

Create documentation for tenant teams about available shared services:

```yaml
# Add annotations to the shared-services namespaces
kubectl annotate namespace monitoring \
  "docs/description=Prometheus and Grafana monitoring stack" \
  "docs/access=All teams have read-only access to Grafana dashboards" \
  "docs/url=https://grafana.cluster.example.com"

kubectl annotate namespace logging \
  "docs/description=Loki log aggregation with Promtail agents" \
  "docs/access=Teams can view logs from their own namespaces" \
  "docs/url=https://grafana.cluster.example.com/explore"

kubectl annotate namespace ingress \
  "docs/description=Nginx ingress controller for external traffic routing" \
  "docs/access=Teams create Ingress resources in their own namespaces" \
  "docs/usage=Use ingress class 'nginx' in your Ingress resources"
```

## Step 11: Maintain the Shared Services Project

Schedule regular maintenance tasks:

```bash
#!/bin/bash
# shared-services-health-check.sh

echo "=== Shared Services Health Check ==="
echo "Date: $(date)"

# Check all deployments in shared namespaces
for ns in monitoring logging ingress cert-manager shared-databases; do
  echo ""
  echo "--- Namespace: $ns ---"
  kubectl get deployments -n $ns -o custom-columns=NAME:.metadata.name,READY:.status.readyReplicas,DESIRED:.spec.replicas

  # Check for pods not in Running state
  unhealthy=$(kubectl get pods -n $ns --field-selector=status.phase!=Running,status.phase!=Succeeded --no-headers 2>/dev/null | wc -l)
  if [ "$unhealthy" -gt 0 ]; then
    echo "  WARNING: $unhealthy unhealthy pods"
    kubectl get pods -n $ns --field-selector=status.phase!=Running,status.phase!=Succeeded
  fi
done

# Check resource usage
echo ""
echo "--- Resource Usage ---"
for ns in monitoring logging ingress cert-manager shared-databases; do
  kubectl top pods -n $ns --no-headers 2>/dev/null | awk -v ns=$ns '{cpu+=$2; mem+=$3} END {printf "  %s: CPU=%dm, Memory=%dMi\n", ns, cpu, mem}'
done
```

## Best Practices

- **Separate from the System project**: Do not deploy custom shared services in the kube-system or cattle-system namespaces.
- **Platform team ownership**: The shared services project should be owned and maintained by the platform team.
- **Read-only access for tenants**: Give tenant teams read-only access for debugging but not the ability to modify shared infrastructure.
- **Network policies are essential**: Carefully configure network policies so tenants can reach shared services but shared services cannot reach arbitrary tenant resources.
- **Resource quotas for shared services**: Set appropriate quotas on the shared services project to prevent monitoring or logging from consuming excessive resources.
- **High availability**: Deploy shared services with multiple replicas and anti-affinity rules.
- **Document everything**: Maintain clear documentation about what shared services are available and how to use them.
- **Separate upgrade cycles**: Upgrade shared services independently from tenant workloads.

## Conclusion

A shared services project in Rancher centralizes cross-cutting infrastructure while maintaining proper isolation and access control. By creating dedicated namespaces for monitoring, logging, ingress, and other shared tools, configuring appropriate network policies, and limiting tenant access to read-only, you build a maintainable multi-tenant platform. The platform team manages shared infrastructure, and tenant teams consume it through well-defined interfaces.
