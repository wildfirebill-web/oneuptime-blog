# How to Automate Compliance Reporting in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, compliance, reporting, cis, automation, pci-dss

Description: A guide to automating compliance reporting in Rancher for CIS benchmarks, PCI DSS, HIPAA, and custom compliance frameworks using scheduled scans and report generation.

## Overview

Compliance reporting is a critical requirement for organizations in regulated industries. Manually generating compliance reports is time-consuming and error-prone. Rancher provides built-in CIS scanning capabilities, and combined with NeuVector's compliance features and custom reporting scripts, you can automate the generation, aggregation, and distribution of compliance reports. This guide covers building an automated compliance reporting pipeline.

## Automated CIS Benchmark Reports

### Schedule Recurring CIS Scans

```yaml
# Schedule CIS scans for all clusters
apiVersion: cis.cattle.io/v1
kind: ScheduledClusterScan
metadata:
  name: monthly-compliance-scan
  namespace: cis-operator-system
spec:
  enabled: true
  cronSchedule: "0 1 1 * *"   # Monthly on the 1st
  scanConfig:
    scanProfileName: rke2-cis-1.23-hardened
  retentionCount: 13    # 13 months for annual comparison
```

### Collect and Aggregate CIS Results

```python
#!/usr/bin/env python3
# generate-compliance-report.py
import subprocess
import json
import datetime
from typing import List, Dict

def get_scan_results() -> List[Dict]:
    """Get CIS scan results from all clusters"""
    result = subprocess.run(
        ['kubectl', 'get', 'clusterscans', '-A',
         '-o', 'json'],
        capture_output=True, text=True, check=True
    )
    scans = json.loads(result.stdout)
    return scans.get('items', [])


def generate_summary_report(scans: List[Dict]) -> str:
    """Generate a markdown compliance summary report"""
    today = datetime.date.today().isoformat()

    report = f"""# Kubernetes Compliance Report
Generated: {today}

## Executive Summary

"""
    total_pass = 0
    total_fail = 0
    cluster_results = []

    for scan in scans:
        cluster_name = scan.get('metadata', {}).get('namespace', 'unknown')
        summary = scan.get('status', {}).get('summary', {})

        if summary:
            pass_count = summary.get('pass', 0)
            fail_count = summary.get('fail', 0)
            warn_count = summary.get('warn', 0)
            total = summary.get('total', 0)

            pass_pct = (pass_count / total * 100) if total > 0 else 0

            total_pass += pass_count
            total_fail += fail_count

            status_emoji = "✅" if pass_pct >= 90 else "⚠️" if pass_pct >= 80 else "❌"

            cluster_results.append({
                'name': cluster_name,
                'pass': pass_count,
                'fail': fail_count,
                'warn': warn_count,
                'total': total,
                'pass_pct': pass_pct,
                'status': status_emoji
            })

    # Summary table
    report += "| Cluster | Pass | Fail | Warn | Pass % | Status |\n"
    report += "|---------|------|------|------|--------|--------|\n"

    for cr in cluster_results:
        report += f"| {cr['name']} | {cr['pass']} | {cr['fail']} | {cr['warn']} | {cr['pass_pct']:.1f}% | {cr['status']} |\n"

    overall_total = total_pass + total_fail
    overall_pct = (total_pass / overall_total * 100) if overall_total > 0 else 0

    report += f"\n**Overall Pass Rate: {overall_pct:.1f}%**\n"

    return report


def get_failed_controls(scan) -> List[str]:
    """Extract list of failed CIS controls"""
    failed = []
    results = scan.get('status', {}).get('lastRunScanStats', {})
    # Add logic to extract specific failed checks
    return failed


if __name__ == '__main__':
    scans = get_scan_results()
    report = generate_summary_report(scans)

    # Save report
    filename = f"compliance-report-{datetime.date.today().isoformat()}.md"
    with open(filename, 'w') as f:
        f.write(report)

    print(f"Report generated: {filename}")
    print(report)
```

## NeuVector Compliance Automation

### Schedule NeuVector Compliance Scans

```bash
#!/bin/bash
# run-neuvector-compliance.sh

NEUVECTOR_URL="${NEUVECTOR_URL}"

# Authenticate
TOKEN=$(curl -s -k \
  -X POST "${NEUVECTOR_URL}/auth" \
  -d '{"password": {"username": "admin", "password": "${NEUVECTOR_PASS}"}}' \
  | jq -r '.token.token')

# Trigger compliance scan
curl -s -k \
  -X POST "${NEUVECTOR_URL}/v1/compliance/run" \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{"request": {"type": "container", "filters": []}}'

# Wait for scan to complete (poll)
for i in $(seq 1 20); do
  STATUS=$(curl -s -k \
    -H "Authorization: Bearer ${TOKEN}" \
    "${NEUVECTOR_URL}/v1/compliance/last" \
    | jq -r '.report.status')

  if [ "${STATUS}" = "done" ]; then
    break
  fi
  sleep 15
done

# Download report
curl -s -k \
  -H "Authorization: Bearer ${TOKEN}" \
  "${NEUVECTOR_URL}/v1/compliance/report" \
  -o "neuvector-compliance-$(date +%Y%m%d).json"

echo "NeuVector compliance report saved"
```

## Kubernetes Audit Report Generator

```python
#!/usr/bin/env python3
# audit-report.py - Generate access audit report from Kubernetes audit logs

import json
import sys
from collections import defaultdict
from datetime import datetime, timedelta

def analyze_audit_logs(log_file: str, days: int = 30) -> dict:
    """Analyze Kubernetes audit logs for compliance evidence"""

    cutoff = datetime.utcnow() - timedelta(days=days)
    stats = {
        'total_events': 0,
        'users': defaultdict(int),
        'resources': defaultdict(int),
        'verbs': defaultdict(int),
        'sensitive_operations': [],
        'failed_auth': 0
    }

    with open(log_file, 'r') as f:
        for line in f:
            try:
                event = json.loads(line.strip())
            except json.JSONDecodeError:
                continue

            # Parse timestamp
            ts = datetime.fromisoformat(
                event.get('requestReceivedTimestamp', '').replace('Z', '+00:00')
            )
            if ts.replace(tzinfo=None) < cutoff:
                continue

            stats['total_events'] += 1

            # Track users
            user = event.get('user', {}).get('username', 'unknown')
            stats['users'][user] += 1

            # Track resource operations
            resource = event.get('objectRef', {}).get('resource', 'unknown')
            verb = event.get('verb', 'unknown')
            stats['resources'][resource] += 1
            stats['verbs'][verb] += 1

            # Flag sensitive operations
            if resource in ['secrets', 'serviceaccounts'] and verb in ['get', 'list']:
                stats['sensitive_operations'].append({
                    'time': event.get('requestReceivedTimestamp'),
                    'user': user,
                    'resource': resource,
                    'verb': verb
                })

            # Track failed authentications
            status_code = event.get('responseStatus', {}).get('code', 0)
            if status_code in [401, 403]:
                stats['failed_auth'] += 1

    return dict(stats)


if __name__ == '__main__':
    log_file = sys.argv[1] if len(sys.argv) > 1 else '/var/log/kubernetes/audit.log'
    stats = analyze_audit_logs(log_file)

    print(f"Total events (30 days): {stats['total_events']}")
    print(f"Failed auth attempts: {stats['failed_auth']}")
    print(f"Unique users: {len(stats['users'])}")
    print(f"Sensitive operations: {len(stats['sensitive_operations'])}")
```

## Automated Report Distribution

```yaml
# CronJob: Generate and email monthly compliance report
apiVersion: batch/v1
kind: CronJob
metadata:
  name: monthly-compliance-report
  namespace: compliance
spec:
  schedule: "0 9 1 * *"    # 9 AM on the 1st of each month
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: compliance-reporter
          containers:
            - name: reporter
              image: registry.example.com/compliance-tools:latest
              env:
                - name: RANCHER_URL
                  value: "https://rancher.example.com"
                - name: RANCHER_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: rancher-token
                      key: token
                - name: EMAIL_TO
                  value: "ciso@example.com,compliance@example.com"
                - name: S3_BUCKET
                  value: "compliance-reports"
              command:
                - /bin/sh
                - -c
                - |
                  python3 /scripts/generate-compliance-report.py
                  aws s3 cp compliance-report-*.md s3://${S3_BUCKET}/
                  python3 /scripts/send-email.py compliance-report-*.md
          restartPolicy: OnFailure
```

## Compliance Dashboard in Grafana

```yaml
# ConfigMap with Grafana dashboard for compliance metrics
apiVersion: v1
kind: ConfigMap
metadata:
  name: compliance-dashboard
  namespace: cattle-monitoring-system
  labels:
    grafana_dashboard: "1"
data:
  compliance.json: |
    {
      "title": "Kubernetes Compliance Dashboard",
      "panels": [
        {
          "title": "CIS Benchmark Pass Rate",
          "type": "stat",
          "targets": [{"expr": "cis_benchmark_pass_percentage"}]
        },
        {
          "title": "Critical CVEs in Production",
          "type": "stat",
          "targets": [{"expr": "neuvector_critical_cves_production"}]
        }
      ]
    }
```

## Conclusion

Automating compliance reporting in Rancher transforms what would be a manual, time-consuming process into a reliable, scheduled workflow. Automated CIS benchmark scans, NeuVector compliance checks, and Kubernetes audit log analysis together provide comprehensive compliance evidence. Scheduling monthly report generation, distributing reports to stakeholders, and maintaining a compliance dashboard in Grafana ensures that your compliance posture is always visible and actionable. Always map your automated checks back to specific regulatory requirements (PCI DSS, HIPAA, FedRAMP) to demonstrate coverage to auditors.
