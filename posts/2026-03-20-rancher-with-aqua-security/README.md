# Integrating Rancher with Aqua Security

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Aqua Security, Kubernetes, Container Security, DevSecOps

Description: Learn how to integrate Aqua Security with Rancher-managed Kubernetes clusters to enforce container image scanning, runtime protection, and compliance policies.

## What is Aqua Security?

Aqua Security is a cloud-native security platform that provides:

- **Image scanning** — Detect vulnerabilities in container images before deployment
- **Runtime protection** — Monitor and block suspicious container behavior
- **Compliance** — Enforce CIS benchmarks and custom security policies
- **Network policies** — Micro-segmentation and firewall rules for containers

## Architecture Overview

```
Rancher → Kubernetes Cluster → Aqua Server (Enforcer Pods)
                              ↓
                    Image Registry → Aqua Scanner
```

## Prerequisites

- Rancher 2.6+ with a managed Kubernetes cluster
- Aqua Security platform (SaaS or self-hosted)
- `kubectl` access to the cluster
- Aqua license and credentials

## Step 1: Create an Aqua Namespace

```bash
kubectl create namespace aqua
```

## Step 2: Create Aqua Credentials Secret

```bash
kubectl create secret docker-registry aqua-registry \
  --docker-server=registry.aquasec.com \
  --docker-username=your-aqua-username \
  --docker-password=your-aqua-password \
  --namespace aqua
```

## Step 3: Deploy Aqua Server via Helm

In Rancher, navigate to **Apps → Charts** or use Helm directly:

```bash
helm repo add aqua-helm https://helm.aquasec.com
helm repo update

helm install aqua aqua-helm/server \
  --namespace aqua \
  --set imageCredentials.username=your-username \
  --set imageCredentials.password=your-password \
  --set db.external.enabled=true \
  --set db.external.host=your-postgres-host \
  --set db.external.port=5432 \
  --set db.external.name=aqua \
  --set db.external.user=aqua \
  --set db.external.password=your-db-password
```

## Step 4: Deploy Aqua Enforcers

Enforcers run as DaemonSets on every node to provide runtime protection:

```bash
helm install aqua-enforcer aqua-helm/enforcer \
  --namespace aqua \
  --set enforcerToken=your-enforcer-token \
  --set gate.host=aqua-gateway.aqua.svc.cluster.local \
  --set gate.port=8443
```

## Step 5: Configure Image Assurance Policies

In the Aqua Security console:

1. Navigate to **Policies → Image Assurance**
2. Create a new policy with rules such as:
   - Block images with critical CVEs
   - Block images with no scan
   - Require specific base images

## Step 6: Enable Admission Control

Aqua's Admission Controller webhooks block non-compliant pods at deployment time:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: aqua-webhook
webhooks:
  - name: imageassurance.aquasec.com
    clientConfig:
      service:
        name: aqua-webhook-service
        namespace: aqua
        path: "/validate"
    rules:
      - operations: ["CREATE"]
        resources: ["pods"]
```

## Viewing Scan Results in Rancher

After integration, you can view Aqua scan results from the Rancher UI:

1. Go to your cluster → **Workloads**
2. Look for the Aqua Security annotations on pods
3. Click the Aqua icon to view image scan details

## Runtime Policies

Create a runtime policy to block suspicious behavior:

```json
{
  "name": "block-shell-access",
  "description": "Block shell access in production containers",
  "block_container_exec": true,
  "block_root_user": true,
  "enable_fork_guard": true
}
```

## Monitoring and Alerting

Configure Aqua to send alerts to Slack or email:

1. Aqua Console → **Administration → Integrations**
2. Add a Slack webhook for security events
3. Configure alert severity thresholds

## Best Practices

1. **Scan images before pushing** to the registry using `aquasec scan` in CI/CD
2. **Enable admission control** to prevent unscanned images from running
3. **Use least-privilege runtime policies** — block shell access in production
4. **Integrate with Rancher RBAC** so only authorized users can change security policies
5. **Set up alerts** for critical CVEs and policy violations

## Conclusion

Integrating Aqua Security with Rancher provides a comprehensive security layer for your Kubernetes workloads. From image scanning in CI/CD to runtime behavioral monitoring, Aqua enforces security controls at every stage of the container lifecycle without disrupting developer workflows.
