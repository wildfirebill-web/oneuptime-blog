# How to Set Up Image Scanning in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Security, Image Scanning

Description: Learn how to set up container image vulnerability scanning in Rancher-managed clusters using Trivy and other scanning tools.

Container images can contain known vulnerabilities in their base OS packages, language libraries, and application dependencies. Scanning images before and during deployment helps you identify and mitigate these risks. This guide covers setting up image scanning in Rancher-managed clusters.

## Prerequisites

- Rancher v2.5 or later
- kubectl access with admin privileges
- Helm 3 installed
- A container registry accessible from the cluster

## Step 1: Deploy Trivy as a Vulnerability Scanner

Trivy is an open-source vulnerability scanner that integrates well with Kubernetes. Install it using Helm:

```bash
helm repo add aquasecurity https://aquasecurity.github.io/helm-charts/
helm repo update

helm install trivy-operator aquasecurity/trivy-operator \
  -n trivy-system \
  --create-namespace \
  --set trivy.ignoreUnfixed=true
```

Verify the installation:

```bash
kubectl get pods -n trivy-system
```

## Step 2: Configure Trivy Operator for Automatic Scanning

The Trivy Operator automatically scans workloads running in the cluster and generates VulnerabilityReport resources.

Configure scan settings:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: trivy-operator
  namespace: trivy-system
data:
  trivy.severity: "CRITICAL,HIGH,MEDIUM"
  trivy.timeout: "10m0s"
  scanJob.tolerations: "[]"
  vulnerabilityReports.scanner: "Trivy"
  configAuditReports.scanner: "Trivy"
```

Apply the configuration:

```bash
kubectl apply -f trivy-config.yaml
```

## Step 3: View Vulnerability Reports

After the operator scans your workloads, view the results:

```bash
kubectl get vulnerabilityreports -A
```

Get detailed results for a specific workload:

```bash
kubectl get vulnerabilityreport -n production \
  -l trivy-operator.resource.name=my-deployment -o yaml
```

Summarize vulnerabilities across the cluster:

```bash
kubectl get vulnerabilityreports -A -o json | \
  jq '[.items[].report.vulnerabilities[] | .severity] | group_by(.) | map({(.[0]): length}) | add'
```

## Step 4: Scan Images Before Deployment

Install the Trivy CLI on your workstation or CI/CD pipeline:

```bash
# Install Trivy CLI
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
```

Scan an image before deploying:

```bash
trivy image nginx:1.25
trivy image --severity HIGH,CRITICAL your-registry/your-app:latest
```

Integrate into CI/CD pipelines:

```yaml
# GitLab CI example
scan_image:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

## Step 5: Set Up Admission Control for Image Scanning

Prevent deployment of images with critical vulnerabilities using an admission webhook. Deploy the Trivy admission controller:

```bash
helm install trivy-admission aquasecurity/trivy-operator \
  -n trivy-system \
  --set admissionController.enabled=true \
  --set admissionController.failurePolicy=Fail \
  --set admissionController.severity="CRITICAL"
```

This blocks any pod that uses an image with critical vulnerabilities from being deployed.

## Step 6: Configure Private Registry Scanning

If your images are in a private registry, configure Trivy with registry credentials:

```bash
kubectl create secret docker-registry registry-creds \
  -n trivy-system \
  --docker-server=your-registry.example.com \
  --docker-username=scanner \
  --docker-password=YOUR_PASSWORD
```

Update the Trivy Operator configuration:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: trivy-operator-trivy-config
  namespace: trivy-system
type: Opaque
stringData:
  TRIVY_USERNAME: scanner
  TRIVY_PASSWORD: YOUR_PASSWORD
  TRIVY_NON_SSL: "false"
```

## Step 7: Monitor Scanning Results in Rancher

View scan results through the Rancher UI by navigating to the cluster and checking for security-related resources. You can also set up a dashboard using Grafana:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: trivy-dashboard
  namespace: cattle-monitoring-system
  labels:
    grafana_dashboard: "1"
data:
  trivy-dashboard.json: |
    {
      "dashboard": {
        "title": "Trivy Vulnerability Dashboard",
        "panels": [
          {
            "title": "Vulnerabilities by Severity",
            "type": "piechart"
          }
        ]
      }
    }
```

## Step 8: Set Up Alerting for Critical Vulnerabilities

Create alerts when critical vulnerabilities are detected:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: trivy-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: image-scanning
    rules:
    - alert: CriticalVulnerabilityFound
      expr: |
        trivy_image_vulnerabilities{severity="Critical"} > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Critical vulnerability found in image {{ $labels.image_repository }}"
    - alert: HighVulnerabilityCount
      expr: |
        trivy_image_vulnerabilities{severity="High"} > 10
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "More than 10 high vulnerabilities in {{ $labels.image_repository }}"
```

## Step 9: Generate Compliance Reports

Export vulnerability data for compliance reporting:

```bash
# Export all vulnerability reports as JSON
kubectl get vulnerabilityreports -A -o json > vulnerability-report.json

# Generate a summary report
kubectl get vulnerabilityreports -A -o custom-columns=\
NAMESPACE:.metadata.namespace,\
NAME:.metadata.labels.trivy-operator\\.resource\\.name,\
CRITICAL:.report.summary.criticalCount,\
HIGH:.report.summary.highCount,\
MEDIUM:.report.summary.mediumCount
```

## Step 10: Schedule Regular Full Scans

Configure the operator to rescan all workloads periodically:

```bash
helm upgrade trivy-operator aquasecurity/trivy-operator \
  -n trivy-system \
  --set operator.scanJobTimeout=15m \
  --set operator.scanJobsConcurrentLimit=3 \
  --set compliance.cron="0 2 * * *"
```

## Best Practices

- Scan images in your CI/CD pipeline before pushing to the registry.
- Block deployment of images with critical vulnerabilities using admission controllers.
- Regularly update the vulnerability database for accurate scanning.
- Set up alerts for newly discovered critical vulnerabilities.
- Maintain a vulnerability remediation SLA (e.g., critical within 24 hours, high within 7 days).
- Use image signing and verification to ensure only scanned images are deployed.

## Conclusion

Container image scanning is a critical component of Kubernetes security. By deploying the Trivy Operator in your Rancher-managed clusters, scanning images in CI/CD pipelines, and enforcing policies through admission controllers, you can detect and prevent vulnerable images from running in production. Combined with monitoring and alerting, image scanning provides continuous visibility into your application security posture.
