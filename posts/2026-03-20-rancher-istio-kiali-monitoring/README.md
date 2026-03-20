# How to Monitor Istio Traffic with Kiali in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Istio, Kiali, Observability, Service Mesh

Description: A guide to installing and using Kiali to visualize, monitor, and manage Istio service mesh traffic in Rancher-managed Kubernetes clusters.

Kiali is the official observability console for the Istio service mesh. It provides a visual representation of your service mesh topology, displays real-time traffic metrics, highlights configuration issues, and enables you to validate Istio configurations. This guide covers how to install and effectively use Kiali in a Rancher environment.

## Prerequisites

- Istio installed in your Rancher-managed cluster
- Prometheus installed for metrics collection (typically bundled with Istio)
- `kubectl` access to the cluster

## Step 1: Install Kiali via Rancher Apps

Kiali can be installed alongside Istio using the Rancher Apps catalog:

1. Navigate to your cluster in the Rancher UI
2. Go to **Apps** → **Charts**
3. Search for **Kiali**
4. Click **Install** and configure the Helm values

Alternatively, install via Helm directly:

```bash
# Add the Kiali Helm repository
helm repo add kiali https://kiali.org/helm-charts
helm repo update

# Install Kiali operator
helm install \
  --namespace kiali-operator \
  --create-namespace \
  kiali-operator \
  kiali/kiali-operator

# Create a Kiali custom resource to deploy the Kiali server
kubectl apply -f - <<EOF
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  name: kiali
  namespace: istio-system
spec:
  auth:
    strategy: anonymous
  deployment:
    accessible_namespaces:
    - "**"
  external_services:
    prometheus:
      url: http://prometheus-server.monitoring.svc.cluster.local
    grafana:
      enabled: true
      url: http://grafana.monitoring.svc.cluster.local
EOF
```

## Step 2: Access the Kiali Dashboard

```bash
# Port-forward to access Kiali locally
kubectl port-forward svc/kiali -n istio-system 20001:20001

# Open in your browser
echo "Access Kiali at: http://localhost:20001/kiali"

# Alternatively, expose via Istio Gateway
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kiali-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - kiali.example.com
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kiali-vs
  namespace: istio-system
spec:
  hosts:
  - kiali.example.com
  gateways:
  - kiali-gateway
  http:
  - route:
    - destination:
        host: kiali
        port:
          number: 20001
EOF
```

## Step 3: Understanding the Kiali Service Graph

The **Service Graph** (also called the **Traffic Graph**) is Kiali's most powerful feature. It shows:

- **Nodes**: Services, workloads, and applications
- **Edges**: Traffic flowing between services with error rates and latency
- **Lock icons**: mTLS status for each connection
- **Colored edges**: Green (healthy), Yellow (degraded), Red (errors)

To view the service graph:
1. Open Kiali and navigate to **Graph** in the left menu
2. Select your namespace(s) from the dropdown
3. Choose the display options (edges labels, node labels, etc.)
4. Use the time range selector to view historical data

## Step 4: Configure Prometheus for Metrics

Kiali relies on Prometheus for metrics. Ensure the Istio metrics are being collected:

```yaml
# prometheus-scrape-config.yaml - Ensure Istio metrics are scraped
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    scrape_configs:
    # Scrape Istio metrics from Envoy proxies
    - job_name: 'istio-mesh'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: istio-telemetry;prometheus
```

## Step 5: Validate Istio Configuration with Kiali

Kiali can detect and highlight configuration issues:

```bash
# Check Kiali's configuration validation via CLI
kubectl get kiali -n istio-system kiali -o yaml | grep -A10 "validations"

# Or use istioctl which provides similar validation
istioctl analyze -n my-app

# Check for common issues:
# - DestinationRule subsets that don't match any pods
# - VirtualServices that reference non-existent Gateways
# - Services without matching workloads
```

In the Kiali UI:
1. Go to **Services** or **Workloads**
2. Look for yellow warning icons or red error icons
3. Click on the service to see the specific validation messages

## Step 6: View Distributed Traces with Jaeger Integration

Kiali integrates with Jaeger for distributed tracing:

```bash
# Install Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.17/samples/addons/jaeger.yaml

# Update Kiali configuration to point to Jaeger
kubectl patch kiali kiali -n istio-system --type=merge -p '{
  "spec": {
    "external_services": {
      "tracing": {
        "enabled": true,
        "url": "http://tracing.istio-system.svc.cluster.local:16686"
      }
    }
  }
}'
```

## Step 7: Generate Traffic for Visualization

```bash
# Generate test traffic to see in Kiali
# If using the Bookinfo sample app:
export GATEWAY_URL=http://$(kubectl -n istio-system get service \
  istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Send traffic every second for 1 minute
for i in $(seq 1 60); do
  curl -s -o /dev/null "$GATEWAY_URL/productpage"
  sleep 1
done
```

## Conclusion

Kiali transforms Istio's raw telemetry data into actionable insights through its visual service graph, configuration validation, and distributed tracing integration. By combining Kiali with Prometheus and Jaeger in your Rancher environment, you get a comprehensive observability platform that makes it easy to understand service dependencies, identify bottlenecks, and troubleshoot issues in your microservice architecture.
