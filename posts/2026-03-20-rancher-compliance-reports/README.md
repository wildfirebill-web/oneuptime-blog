# How to Generate Compliance Reports in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Compliance, Reporting, CIS, Security

Description: Learn how to generate, export, and present compliance reports from Rancher's CIS scanning tool for audits and security reviews.

Compliance reporting is a critical requirement for organizations operating Kubernetes clusters in regulated industries. Rancher's CIS scanning integration provides detailed compliance reports that can be exported in various formats for auditors and stakeholders. This guide covers how to generate comprehensive compliance reports from Rancher.

## Prerequisites

- Rancher with CIS Benchmark app installed and scans configured
- Completed CIS scans with results
- Cluster access permissions
- Python 3 installed (for report processing scripts)

## Step 1: Access Scan Reports in Rancher UI

1. Navigate to your cluster in the Rancher UI
2. Go to **CIS Benchmark** → **Scans**
3. Click on a completed scan to view results
4. The report shows:
   - Overall pass/fail summary
   - Detailed results by CIS section
   - Remediation guidance for failed checks
5. Click **Download** to export the report

## Step 2: Retrieve Reports via kubectl

```bash
# List all scan reports
kubectl get clusterscanreport -A

# Get the most recent scan report name
LATEST_REPORT=$(kubectl get clusterscanreport \
  --sort-by='.metadata.creationTimestamp' \
  -o jsonpath='{.items[-1].metadata.name}')

echo "Latest report: $LATEST_REPORT"

# Export the full report as JSON
kubectl get clusterscanreport $LATEST_REPORT \
  -o jsonpath='{.spec.reportJSON}' > compliance-report.json

echo "Report exported to compliance-report.json"
```

## Step 3: Generate a Summary Report

```python
#!/usr/bin/env python3
# generate-compliance-summary.py - Generate a human-readable compliance summary

import json
import sys
from datetime import datetime

def generate_summary(report_file):
    with open(report_file) as f:
        report = json.load(f)

    total_pass = 0
    total_fail = 0
    total_skip = 0
    total_warn = 0
    failed_checks = []

    for result in report.get('results', []):
        for check in result.get('checks', []):
            state = check.get('state', '')
            if state == 'pass':
                total_pass += 1
            elif state == 'fail':
                total_fail += 1
                failed_checks.append({
                    'id': check.get('id'),
                    'description': check.get('description'),
                    'remediation': check.get('remediation', 'No remediation provided')
                })
            elif state == 'skip':
                total_skip += 1
            elif state == 'warn':
                total_warn += 1

    total = total_pass + total_fail + total_skip + total_warn
    pass_rate = (total_pass / total * 100) if total > 0 else 0

    print("=" * 60)
    print("CIS KUBERNETES BENCHMARK COMPLIANCE REPORT")
    print(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print("=" * 60)
    print(f"\nSUMMARY")
    print(f"  Total Checks: {total}")
    print(f"  Passed:       {total_pass} ({pass_rate:.1f}%)")
    print(f"  Failed:       {total_fail}")
    print(f"  Skipped:      {total_skip}")
    print(f"  Warnings:     {total_warn}")
    print(f"\nCOMPLIANCE STATUS: {'COMPLIANT' if total_fail == 0 else 'NON-COMPLIANT'}")

    if failed_checks:
        print(f"\nFAILED CHECKS ({len(failed_checks)} items requiring remediation):")
        print("-" * 60)
        for check in failed_checks:
            print(f"\n[{check['id']}] {check['description']}")
            print(f"  Remediation: {check['remediation'][:200]}...")

if __name__ == '__main__':
    generate_summary(sys.argv[1] if len(sys.argv) > 1 else 'compliance-report.json')
```

```bash
# Run the summary generator
python3 generate-compliance-summary.py compliance-report.json
```

## Step 4: Generate an HTML Compliance Report

```python
#!/usr/bin/env python3
# generate-html-report.py - Generate an HTML compliance report

import json
from datetime import datetime

def generate_html_report(report_file, output_file='compliance-report.html'):
    with open(report_file) as f:
        report = json.load(f)

    checks_by_state = {'pass': [], 'fail': [], 'skip': [], 'warn': []}

    for result in report.get('results', []):
        for check in result.get('checks', []):
            state = check.get('state', 'skip')
            checks_by_state[state].append(check)

    html = f"""<!DOCTYPE html>
<html>
<head>
    <title>CIS Compliance Report</title>
    <style>
        body {{ font-family: Arial, sans-serif; margin: 20px; }}
        .pass {{ color: green; }}
        .fail {{ color: red; }}
        .skip {{ color: gray; }}
        .warn {{ color: orange; }}
        table {{ border-collapse: collapse; width: 100%; }}
        th, td {{ border: 1px solid #ddd; padding: 8px; text-align: left; }}
        th {{ background-color: #4CAF50; color: white; }}
        tr:nth-child(even) {{ background-color: #f2f2f2; }}
    </style>
</head>
<body>
    <h1>CIS Kubernetes Benchmark Compliance Report</h1>
    <p>Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}</p>

    <h2>Summary</h2>
    <table>
        <tr>
            <th>Status</th>
            <th>Count</th>
        </tr>
        <tr class="pass"><td>Pass</td><td>{len(checks_by_state['pass'])}</td></tr>
        <tr class="fail"><td>Fail</td><td>{len(checks_by_state['fail'])}</td></tr>
        <tr class="skip"><td>Skip</td><td>{len(checks_by_state['skip'])}</td></tr>
        <tr class="warn"><td>Warn</td><td>{len(checks_by_state['warn'])}</td></tr>
    </table>

    <h2>Failed Checks</h2>
    <table>
        <tr>
            <th>Check ID</th>
            <th>Description</th>
            <th>Remediation</th>
        </tr>"""

    for check in checks_by_state['fail']:
        html += f"""
        <tr>
            <td>{check.get('id', '')}</td>
            <td>{check.get('description', '')}</td>
            <td>{check.get('remediation', 'N/A')[:300]}</td>
        </tr>"""

    html += """
    </table>
</body>
</html>"""

    with open(output_file, 'w') as f:
        f.write(html)

    print(f"HTML report generated: {output_file}")

generate_html_report('compliance-report.json')
```

## Step 5: Schedule Automated Report Generation

```bash
# Create a CronJob to generate and store reports
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: compliance-report-generator
  namespace: cattle-cis-system
spec:
  # Generate report weekly on Monday at 8 AM
  schedule: "0 8 * * 1"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: cis-report-sa
          containers:
          - name: report-generator
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              REPORT=$(kubectl get clusterscanreport \
                --sort-by='.metadata.creationTimestamp' \
                -o jsonpath='{.items[-1].metadata.name}')
              kubectl get clusterscanreport \$REPORT \
                -o jsonpath='{.spec.reportJSON}' \
                > /reports/compliance-\$(date +%Y%m%d).json
            volumeMounts:
            - name: reports-volume
              mountPath: /reports
          restartPolicy: OnFailure
          volumes:
          - name: reports-volume
            persistentVolumeClaim:
              claimName: compliance-reports-pvc
EOF
```

## Conclusion

Generating comprehensive compliance reports from Rancher's CIS scanning tool provides the documentation needed for security audits and ongoing compliance monitoring. By automating report generation and combining JSON exports with custom reporting scripts, you can create audit-ready compliance documentation that satisfies regulatory requirements. These reports serve as evidence of your organization's commitment to Kubernetes security best practices.
