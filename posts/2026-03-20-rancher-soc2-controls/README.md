# How to Set Up SOC 2 Controls with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, SOC2, Compliance, Security, Audit

Description: A guide to implementing SOC 2 Trust Services Criteria controls for Kubernetes infrastructure managed by Rancher.

SOC 2 (System and Organization Controls 2) is an auditing framework developed by the AICPA that evaluates an organization's security controls across five Trust Services Criteria. For organizations running Kubernetes on Rancher, implementing SOC 2 controls requires careful configuration of authentication, access control, monitoring, and availability measures. This guide maps Rancher capabilities to SOC 2 requirements.

## Prerequisites

- Rancher v2.6+ managing production Kubernetes clusters
- Understanding of SOC 2 Trust Services Criteria
- Security and compliance team involvement
- Monitoring infrastructure (Prometheus, Grafana, alerting)

## SOC 2 Trust Services Criteria Overview

| Criteria | Code | Focus Area |
|---|---|---|
| Security | CC | Protection against unauthorized access |
| Availability | A | System availability as per SLA |
| Processing Integrity | PI | Complete, valid, accurate processing |
| Confidentiality | C | Protection of confidential information |
| Privacy | P | Personal information collection and use |

## CC6: Logical and Physical Access Controls

### Authentication and Authorization

```yaml
# rancher-auth-config.yaml - Configure LDAP/AD authentication for Rancher
# This is configured in Rancher UI under Global Settings > Authentication
# Key requirements for SOC 2:

# 1. Multi-Factor Authentication (MFA)
# Enable in Rancher UI: Global -> Security -> Authentication
# Rancher supports SAML providers with MFA (Okta, Azure AD, etc.)

# 2. Role-Based Access Control
# Create least-privilege roles for different teams
apiVersion: management.cattle.io/v3
kind: GlobalRole
metadata:
  name: security-auditor
rules:
  # Read-only access for auditors
  - apiGroups: ["management.cattle.io"]
    resources: ["clusters", "nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["cis.cattle.io"]
    resources: ["clusterscans", "clusterscanreports"]
    verbs: ["get", "list", "watch"]
```

```bash
# Audit access: who has access to what
kubectl get clusterrolebindings -A -o json | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
print('=== Cluster Role Bindings ===')
for item in data['items']:
    role = item['roleRef']['name']
    subjects = item.get('subjects', [])
    for s in subjects:
        print(f\"{role}: {s.get('kind')}/{s.get('name')} (ns: {s.get('namespace', 'cluster')})\")
"

# Review and document all service account permissions
kubectl get serviceaccounts -A | grep -v default | grep -v kube
```

## CC7: System Operations Monitoring

### Logging and Monitoring Setup

```yaml
# prometheus-rules.yaml - SOC 2 required monitoring alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: soc2-security-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: soc2-cc7
    rules:
    # Alert on unauthorized access attempts
    - alert: UnauthorizedAPIAccess
      expr: |
        increase(apiserver_request_total{code=~"401|403"}[5m]) > 10
      for: 1m
      labels:
        severity: warning
        soc2: "CC7.2"
      annotations:
        summary: "High rate of unauthorized API access attempts"

    # Alert on node unavailability
    - alert: NodeDown
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
      for: 5m
      labels:
        severity: critical
        soc2: "A1.1"
      annotations:
        summary: "Node is not ready: {{ $labels.node }}"

    # Alert on pod restart storms (availability)
    - alert: PodCrashLooping
      expr: |
        increase(kube_pod_container_status_restarts_total[1h]) > 5
      for: 5m
      labels:
        severity: warning
        soc2: "A1.1"
      annotations:
        summary: "Pod is crash looping: {{ $labels.pod }}"
```

## CC8: Change Management

### Audit Trail for Configuration Changes

```bash
# Enable Kubernetes audit logging for change tracking
# This requires API server configuration (see STIG guide for full config)

# Configure audit policy for SOC 2
cat > /etc/kubernetes/soc2-audit-policy.yaml << 'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all changes to security-sensitive resources
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
    - group: "rbac.authorization.k8s.io"
      resources: ["*"]
    - group: ""
      resources: ["secrets", "serviceaccounts"]
    - group: "apps"
      resources: ["deployments", "daemonsets", "statefulsets"]

  # Log authentication events
  - level: Request
    verbs: ["*"]
    users: ["system:anonymous"]

  # Log all other activities at metadata level
  - level: Metadata
EOF
```

## A1: Availability Commitments

### High Availability Configuration

```yaml
# Check and ensure cluster HA requirements
# For Rancher-managed RKE2 clusters:

# 1. Multi-master setup (3+ control plane nodes)
# 2. etcd clustering with snapshots
# 3. Worker node redundancy

# Configure automatic etcd backups (SOC 2 A1.2)
# /etc/rancher/rke2/config.yaml additions:
# etcd-snapshot-schedule-cron: "0 */6 * * *"
# etcd-snapshot-retention: 10

# Verify backup configuration
kubectl exec -n kube-system -l component=etcd \
  -- etcdctl snapshot status /var/lib/rancher/rke2/server/db/snapshots/ 2>/dev/null || \
  echo "Check /var/lib/rancher/rke2/server/db/snapshots/ on control plane nodes"
```

## C1: Confidentiality Controls

### Secrets Management

```bash
# Ensure secrets are encrypted at rest
kubectl get secrets -n kube-system -o yaml | grep "type:"

# Check if encryption provider is configured
# (See STIG guide for encryption-config.yaml setup)

# Use external secrets management for sensitive data
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace

# Create a SecretStore pointing to your secrets manager (Vault, AWS SM, etc.)
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "rancher-secrets"
EOF
```

## SOC 2 Evidence Collection

```bash
# Script to collect SOC 2 evidence
cat > /tmp/collect-soc2-evidence.sh << 'EOF'
#!/bin/bash
EVIDENCE_DIR="/tmp/soc2-evidence-$(date +%Y%m%d)"
mkdir -p $EVIDENCE_DIR

echo "Collecting SOC 2 evidence..."

# CC6: Access controls
kubectl get clusterrolebindings -o yaml > $EVIDENCE_DIR/clusterrolebindings.yaml
kubectl get roles -A -o yaml > $EVIDENCE_DIR/roles.yaml
kubectl get rolebindings -A -o yaml > $EVIDENCE_DIR/rolebindings.yaml

# CC7: Monitoring configuration
kubectl get prometheusrule -A -o yaml > $EVIDENCE_DIR/prometheus-rules.yaml
kubectl get alertmanagerconfig -A -o yaml > $EVIDENCE_DIR/alertmanager-config.yaml

# A1: Availability - node status
kubectl get nodes -o wide > $EVIDENCE_DIR/node-status.txt
kubectl get pods -A | grep -v Running > $EVIDENCE_DIR/non-running-pods.txt

# CIS scan results
kubectl get clusterscanreport -A -o yaml > $EVIDENCE_DIR/cis-scan-reports.yaml

echo "Evidence collected in: $EVIDENCE_DIR"
tar -czf soc2-evidence-$(date +%Y%m%d).tar.gz $EVIDENCE_DIR
echo "Archive created: soc2-evidence-$(date +%Y%m%d).tar.gz"
EOF

chmod +x /tmp/collect-soc2-evidence.sh
```

## Conclusion

Implementing SOC 2 controls in a Rancher environment involves configuring authentication, access control, audit logging, monitoring, and availability measures across your Kubernetes clusters. The key to SOC 2 compliance is not just implementing technical controls, but also documenting them and being able to demonstrate their effectiveness to auditors. Rancher's built-in features like CIS scanning, RBAC management, and integration with monitoring tools make it well-suited for organizations pursuing SOC 2 certification.
