# How to Schedule Recurring CIS Scans in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, CIS, Security, Compliance, Automation

Description: Learn how to configure recurring CIS benchmark scans in Rancher to continuously monitor your Kubernetes cluster security posture on a schedule.

Continuous compliance monitoring requires running CIS scans regularly, not just on-demand. Rancher supports scheduled CIS scans using cron expressions, enabling automated security compliance checks that run daily, weekly, or on any custom schedule. This guide covers how to set up and manage recurring CIS scans.

## Prerequisites

- Rancher with CIS Benchmark app installed
- Cluster Owner or Cluster Admin permissions
- Understanding of cron expressions
- Notification channels configured in Rancher (optional, for alerts)

## Step 1: Configure a Scheduled Scan via Rancher UI

1. Navigate to your cluster in the Rancher UI
2. Go to **CIS Benchmark** → **Scans**
3. Click **Scan** to create a new scan
4. Toggle on **Scheduled Scan**
5. Configure the cron schedule:
   - **Daily at midnight**: `0 0 * * *`
   - **Weekly on Monday at 2 AM**: `0 2 * * 1`
   - **Every 6 hours**: `0 */6 * * *`
6. Set the **Retention Count** (how many reports to keep)
7. Click **Create**

## Step 2: Configure a Scheduled Scan via kubectl

```yaml
# daily-cis-scan.yaml - Run CIS scan daily at midnight
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: daily-cis-scan
  namespace: default
spec:
  scanProfileName: rke2-cis-1.6-profile-hardened
  scheduledScanConfig:
    # Run at midnight every day
    cronSchedule: "0 0 * * *"
    # Keep the last 7 reports
    retentionCount: 7
```

```bash
kubectl apply -f daily-cis-scan.yaml

# Verify the scheduled scan was created
kubectl describe clusterscan daily-cis-scan
```

## Step 3: Configure Multiple Scan Schedules

Run different scans at different frequencies:

```yaml
# hourly-quick-scan.yaml - Frequent scan for critical namespaces
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: hourly-critical-scan
spec:
  scanProfileName: rke2-cis-1.6-profile-hardened
  scheduledScanConfig:
    # Run every hour
    cronSchedule: "0 * * * *"
    retentionCount: 24
---
# weekly-full-scan.yaml - Comprehensive weekly scan
apiVersion: cis.cattle.io/v1
kind: ClusterScan
metadata:
  name: weekly-comprehensive-scan
spec:
  scanProfileName: rke2-cis-1.6-profile-hardened
  scheduledScanConfig:
    # Run every Sunday at 1 AM
    cronSchedule: "0 1 * * 0"
    retentionCount: 52
```

## Step 4: Monitor Scheduled Scan Execution

```bash
# List all cluster scans and their last run time
kubectl get clusterscan -A

# Check the next scheduled run time
kubectl get clusterscan daily-cis-scan \
  -o jsonpath='{.status.nextScanAt}'

# View the last scan run time
kubectl get clusterscan daily-cis-scan \
  -o jsonpath='{.status.lastRunAt}'

# Check scan history
kubectl get clusterscanreport -A --sort-by='.metadata.creationTimestamp'
```

## Step 5: Set Up Alerts for Scan Failures

Configure notifications when scans fail or find new issues:

```bash
# First, configure a Rancher notifier (Slack, PagerDuty, etc.)
# Navigate to: Cluster -> Monitoring -> Notifiers -> Add Notifier

# Create an alert rule for CIS scan failures
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cis-scan-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: cis-scan
    interval: 1m
    rules:
    # Alert when CIS scan failure count increases
    - alert: CISScanFailuresIncreased
      expr: |
        increase(cis_scan_failures_total[1h]) > 5
      for: 0m
      labels:
        severity: warning
      annotations:
        summary: "CIS scan failures increased"
        description: "More than 5 new CIS scan failures in the last hour"
    # Alert when scan hasn't run successfully
    - alert: CISScanNotRunning
      expr: |
        time() - cis_scan_last_success_timestamp > 86400
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "CIS scan has not run successfully in 24 hours"
EOF
```

## Step 6: Automate Remediation Tracking

```bash
# Create a script to compare consecutive scan results
cat > /tmp/compare-scans.sh << 'EOF'
#!/bin/bash
# Compare two CIS scan reports and show regressions

REPORT1=$1
REPORT2=$2

echo "=== CIS Scan Comparison ==="
echo "Previous: $REPORT1"
echo "Current: $REPORT2"
echo ""

# Get failed checks from each report
FAILS1=$(kubectl get clusterscanreport $REPORT1 \
  -o jsonpath='{.spec.reportJSON}' | \
  python3 -c "
import json,sys
r=json.load(sys.stdin)
fails=set()
for result in r.get('results',[]):
    for check in result.get('checks',[]):
        if check['state']=='fail':
            fails.add(check['id'])
print('\n'.join(sorted(fails)))
")

FAILS2=$(kubectl get clusterscanreport $REPORT2 \
  -o jsonpath='{.spec.reportJSON}' | \
  python3 -c "
import json,sys
r=json.load(sys.stdin)
fails=set()
for result in r.get('results',[]):
    for check in result.get('checks',[]):
        if check['state']=='fail':
            fails.add(check['id'])
print('\n'.join(sorted(fails)))
")

echo "=== New Failures (regressions) ==="
comm -13 <(echo "$FAILS1" | sort) <(echo "$FAILS2" | sort)

echo ""
echo "=== Fixed Issues ==="
comm -23 <(echo "$FAILS1" | sort) <(echo "$FAILS2" | sort)
EOF

chmod +x /tmp/compare-scans.sh
```

## Step 7: Integrate with CI/CD Pipeline

```yaml
# .github/workflows/cis-scan.yaml - GitHub Actions workflow to trigger scans
name: CIS Compliance Check

on:
  push:
    branches: [main]
  schedule:
    # Also run on a schedule
    - cron: '0 2 * * *'

jobs:
  cis-scan:
    runs-on: ubuntu-latest
    steps:
    - name: Trigger CIS Scan
      run: |
        # Trigger a new scan via Rancher API
        curl -X POST \
          -H "Authorization: Bearer ${{ secrets.RANCHER_TOKEN }}" \
          -H "Content-Type: application/json" \
          "${{ secrets.RANCHER_URL }}/v1/cis.cattle.io.clusterscans" \
          -d '{
            "metadata": {"name": "ci-scan"},
            "spec": {
              "scanProfileName": "rke2-cis-1.6-profile-hardened"
            }
          }'
```

## Conclusion

Scheduling recurring CIS scans in Rancher provides continuous visibility into your cluster's security posture. By combining automated scans with alerting and automated remediation tracking, you can maintain a proactive security stance and quickly identify when new configuration changes introduce compliance violations. Regular scans are a key component of any mature security and compliance program for Kubernetes environments.
