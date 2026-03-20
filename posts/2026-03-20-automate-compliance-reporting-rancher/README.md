# How to Automate Compliance Reporting in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Compliance, Reporting, CIS, SOC2, Kubernetes, Automation

Description: Automate compliance reporting in Rancher for SOC 2, PCI-DSS, and HIPAA using CIS benchmark scans, RBAC audits, vulnerability reports, and automated report generation for auditors and security teams.

## Introduction

Compliance reporting in Kubernetes environments is time-consuming when done manually. Security teams spend weeks before audits gathering evidence: CIS benchmark results, access control lists, audit logs, vulnerability scan reports, and network policy configurations. Automating this evidence collection and report generation reduces audit preparation time from weeks to minutes.

## Step 1: CIS Benchmark Automated Reports

```bash
# Schedule regular CIS benchmark scans
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cis-monthly-scan
  namespace: cattle-system
spec:
  schedule: "0 2 1 * *"    # First of every month
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cis-report-sa
          containers:
            - name: scanner
              image: myregistry/cis-reporter:latest
              command:
                - /bin/sh
                - -c
                - |
                  # Trigger CIS scan
                  kubectl apply -f - <<SCAN_EOF
                  apiVersion: cis.cattle.io/v1
                  kind: ClusterScan
                  metadata:
                    name: compliance-$(date +%Y%m)
                  spec:
                    scanProfileName: rke2-cis-1.24-profile
                  SCAN_EOF

                  # Wait for completion
                  kubectl wait --for=condition=complete \
                    clusterscan/compliance-$(date +%Y%m) --timeout=1h

                  # Export to S3
                  kubectl get clusterscansummary compliance-$(date +%Y%m) -o json | \
                    aws s3 cp - s3://compliance-reports/cis/$(date +%Y%m).json
          restartPolicy: OnFailure
EOF
```

## Step 2: RBAC Compliance Report

```python
# rbac_compliance_report.py
import subprocess
import json
from datetime import datetime

def generate_rbac_report(cluster_contexts: list) -> dict:
    report = {
        'generated': datetime.utcnow().isoformat(),
        'clusters': {}
    }

    for ctx in cluster_contexts:
        cluster_data = {
            'cluster_admin_bindings': [],
            'privileged_namespaces': [],
            'service_accounts_with_cluster_roles': [],
            'users_without_mfa': []
        }

        # Find cluster-admin bindings
        result = subprocess.run([
            'kubectl', '--context', ctx,
            'get', 'clusterrolebindings',
            '-o', 'json'
        ], capture_output=True, text=True)

        bindings = json.loads(result.stdout)
        for binding in bindings['items']:
            if binding['roleRef']['name'] == 'cluster-admin':
                if not binding['metadata']['name'].startswith('system:'):
                    cluster_data['cluster_admin_bindings'].append({
                        'name': binding['metadata']['name'],
                        'subjects': binding.get('subjects', [])
                    })

        # Find privileged pods
        result = subprocess.run([
            'kubectl', '--context', ctx,
            'get', 'pods', '-A',
            '-o', 'json'
        ], capture_output=True, text=True)

        pods = json.loads(result.stdout)
        for pod in pods['items']:
            for container in pod['spec'].get('containers', []):
                if container.get('securityContext', {}).get('privileged'):
                    cluster_data['privileged_namespaces'].append({
                        'namespace': pod['metadata']['namespace'],
                        'pod': pod['metadata']['name'],
                        'container': container['name']
                    })

        report['clusters'][ctx] = cluster_data

    return report
```

## Step 3: Vulnerability Compliance Summary

```yaml
# CronJob to generate weekly vulnerability compliance report
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vuln-compliance-report
  namespace: trivy-system
spec:
  schedule: "0 7 * * 1"    # Monday mornings
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: reporter
              image: myregistry/compliance-reporter:latest
              command:
                - /bin/sh
                - -c
                - |
                  # Aggregate vulnerability data from Trivy Operator
                  kubectl get vulnerabilityreports -A -o json | \
                    python3 /scripts/generate_vuln_report.py \
                    --format pdf \
                    --output /tmp/vuln-report-$(date +%Y%m%d).pdf

                  # Upload to compliance storage
                  aws s3 cp /tmp/vuln-report-$(date +%Y%m%d).pdf \
                    s3://compliance-reports/vulnerabilities/

                  # Send notification
                  curl -X POST "$SLACK_WEBHOOK" \
                    -d "{\"text\": \"Weekly vulnerability compliance report generated: $(date)\"}"
          restartPolicy: OnFailure
```

## Step 4: Audit Log Report

```python
# audit_log_analysis.py - Analyze K8s audit logs for compliance

def analyze_audit_logs(log_file: str) -> dict:
    """Extract compliance-relevant events from audit log"""
    findings = {
        'privileged_exec': [],
        'secret_access': [],
        'rbac_changes': [],
        'failed_auth': [],
        'admin_activity': []
    }

    with open(log_file) as f:
        for line in f:
            entry = json.loads(line)

            # Track exec into privileged pods
            if (entry.get('verb') == 'create' and
                'exec' in entry.get('requestURI', '')):
                findings['privileged_exec'].append({
                    'timestamp': entry['requestReceivedTimestamp'],
                    'user': entry['user']['username'],
                    'pod': entry['requestURI']
                })

            # Track secret access
            if (entry.get('objectRef', {}).get('resource') == 'secrets' and
                entry.get('verb') in ['get', 'list']):
                findings['secret_access'].append({
                    'timestamp': entry['requestReceivedTimestamp'],
                    'user': entry['user']['username'],
                    'namespace': entry['objectRef'].get('namespace'),
                    'secret': entry['objectRef'].get('name')
                })

            # Track RBAC changes
            if entry.get('objectRef', {}).get('apiGroup') == 'rbac.authorization.k8s.io':
                if entry.get('verb') in ['create', 'update', 'delete']:
                    findings['rbac_changes'].append({
                        'timestamp': entry['requestReceivedTimestamp'],
                        'user': entry['user']['username'],
                        'action': f"{entry['verb']} {entry['objectRef']['resource']}/{entry['objectRef'].get('name')}"
                    })

    return findings
```

## Step 5: Automated Evidence Collection

```bash
#!/bin/bash
# collect_audit_evidence.sh - Gather all compliance evidence

EVIDENCE_DIR="/tmp/audit-evidence-$(date +%Y%m%d)"
mkdir -p "$EVIDENCE_DIR"

echo "Collecting compliance evidence..."

# 1. Cluster configuration
kubectl cluster-info dump > "$EVIDENCE_DIR/cluster-info.json"

# 2. RBAC configuration
kubectl get clusterrolebindings -o json > "$EVIDENCE_DIR/clusterrolebindings.json"
kubectl get rolebindings -A -o json > "$EVIDENCE_DIR/rolebindings.json"

# 3. Network policies
kubectl get networkpolicies -A -o json > "$EVIDENCE_DIR/networkpolicies.json"

# 4. Pod security configuration
kubectl get namespaces -o json > "$EVIDENCE_DIR/namespaces.json"

# 5. Secret encryption status
kubectl get --raw /api/v1/namespaces/kube-system | \
  grep -i "encryption" > "$EVIDENCE_DIR/encryption-status.txt"

# 6. Vulnerability reports
kubectl get vulnerabilityreports -A -o json > "$EVIDENCE_DIR/vulnerability-reports.json"

# 7. CIS scan results
kubectl get clusterscansummaries -o json > "$EVIDENCE_DIR/cis-results.json"

# Package evidence
tar czf "audit-evidence-$(date +%Y%m%d).tar.gz" "$EVIDENCE_DIR"
aws s3 cp "audit-evidence-$(date +%Y%m%d).tar.gz" s3://audit-evidence/

echo "Evidence collected: audit-evidence-$(date +%Y%m%d).tar.gz"
```

## Step 6: Executive Compliance Dashboard

```yaml
# Grafana dashboard panels for executive reporting
# Using data from Trivy Operator + CIS Benchmark

panels:
  - title: "Overall Security Score"
    type: stat
    targets:
      - expr: |
          (sum(cis_benchmark_pass_count) /
           sum(cis_benchmark_total_count)) * 100

  - title: "Critical Vulnerabilities Trend"
    type: timeseries
    targets:
      - expr: sum(trivy_image_vulnerabilities{severity="CRITICAL"})

  - title: "Clusters Passing CIS Benchmark"
    type: gauge
    targets:
      - expr: sum(cis_benchmark_pass_count) / count(kube_node_info)
```

## Compliance Framework Coverage

| Framework | Automated Controls |
|---|---|
| SOC 2 | Access logs, change management, vuln scans |
| PCI-DSS | Network policies, encryption, access control |
| HIPAA | Audit logs, access control, encryption at rest |
| ISO 27001 | RBAC review, patch management, monitoring |

## Conclusion

Automating compliance reporting in Rancher transforms audit preparation from a multi-week manual process into an on-demand click. Schedule CIS benchmark scans monthly, generate RBAC reports weekly, and run vulnerability summaries daily. When auditors arrive, run the evidence collection script to package all required artifacts. The Grafana compliance dashboard provides continuous visibility for security teams, while automated reports deliver consistent, timestamped evidence that satisfies external auditors.
