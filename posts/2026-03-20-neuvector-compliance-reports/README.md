# How to Generate NeuVector Compliance Reports

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Compliance, Reporting, CIS Benchmarks, Kubernetes

Description: Generate comprehensive compliance reports from NeuVector covering CIS Benchmarks, PCI DSS, HIPAA, and other regulatory frameworks for auditors and stakeholders.

## Introduction

NeuVector's compliance reporting capabilities enable you to generate auditor-ready reports that demonstrate your container security posture against regulatory frameworks. This guide covers generating compliance reports via the UI, REST API, and automated scripting for regular reporting cycles.

## Report Types Available

NeuVector can generate reports covering:

- **CIS Docker Benchmark**: Host-level container security
- **CIS Kubernetes Benchmark**: Kubernetes cluster hardening
- **PCI DSS**: Payment card industry requirements
- **HIPAA**: Healthcare data protection controls
- **GDPR**: Data privacy controls
- **NIST 800-190**: Container security guide
- **Custom compliance checks**: Organization-specific requirements

## Prerequisites

- NeuVector with Enforcer running
- Compliance scans completed (run scans if needed)
- NeuVector Manager access

## Step 1: Run Compliance Scans

Before generating reports, ensure fresh scan data:

```bash
# Trigger compliance scan on all nodes
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/bench/host/all" \
  -H "X-Auth-Token: ${TOKEN}"

# Wait for scans to complete
sleep 60

# Check scan status
curl -sk \
  "https://neuvector-manager:8443/v1/bench/host" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.hosts[] | {
    host: .host,
    status: .status,
    scanned: .scanned_at
  }'
```

## Step 2: Export Compliance Report from UI

1. Navigate to **Security Risks** > **Compliance**
2. Select the compliance framework from the dropdown:
   - CIS, PCI, HIPAA, GDPR, NIST
3. Use filters to scope the report:
   - Namespace
   - Node
   - Severity level
4. Click **Export** (top right) to download CSV

## Step 3: Generate Reports via API

```bash
#!/bin/bash
# generate-compliance-report.sh

NV_URL="https://neuvector-manager:8443"
REPORT_DATE=$(date +%Y-%m-%d)
OUTPUT_DIR="/reports/neuvector/${REPORT_DATE}"
mkdir -p "${OUTPUT_DIR}"

# Authenticate
TOKEN=$(curl -sk -X POST "${NV_URL}/v1/auth" \
  -H "Content-Type: application/json" \
  -d '{"password":{"username":"admin","password":"yourpassword"}}' \
  | jq -r '.token.token')

# Generate Docker CIS Benchmark Report
echo "Generating Docker CIS Benchmark report..."
curl -sk "${NV_URL}/v1/bench/host" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq -r '
    "Docker CIS Benchmark Report - '"${REPORT_DATE}"'",
    "=" * 60,
    "",
    "SUMMARY",
    "-" * 40,
    (.hosts[] |
      "Host: \(.host)",
      "  Passed:  \(.passed)",
      "  Warned:  \(.warned)",
      "  Failed:  \(.failed)",
      "  Total:   \(.total)",
      ""
    )' > "${OUTPUT_DIR}/cis-docker-benchmark.txt"

echo "Generating detailed CIS checks..."
for HOST in $(curl -sk "${NV_URL}/v1/bench/host" \
  -H "X-Auth-Token: ${TOKEN}" | jq -r '.hosts[].host'); do

  curl -sk "${NV_URL}/v1/bench/host/${HOST}/docker" \
    -H "X-Auth-Token: ${TOKEN}" | \
    jq -r --arg host "$HOST" '
      "Host: \($host)",
      (.items[] | select(.level != "PASS") |
        "[\(.level)] \(.test_number): \(.description)",
        "  Remediation: \(.remediation)",
        ""
      )' >> "${OUTPUT_DIR}/cis-detailed-findings.txt"
done

echo "Reports saved to ${OUTPUT_DIR}"
```

## Step 4: Generate CSV Compliance Reports

For spreadsheet-based reporting:

```bash
# Generate CSV report of all compliance findings
curl -sk \
  "https://neuvector-manager:8443/v1/bench/host" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq -r '.hosts[] |
    .host as $host |
    .items[]? |
    [$host, .test_number, .description, .level, .category, .remediation] |
    @csv' | \
  awk 'BEGIN{print "\"Host\",\"Check ID\",\"Description\",\"Level\",\"Category\",\"Remediation\""}{print}' \
  > compliance-all-findings.csv

# Filter only failures for executive summary
curl -sk \
  "https://neuvector-manager:8443/v1/bench/host" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq -r '.hosts[] |
    .host as $host |
    .items[]? |
    select(.level == "FAIL") |
    [$host, .test_number, .description, .category, .remediation] |
    @csv' | \
  awk 'BEGIN{print "\"Host\",\"Check ID\",\"Description\",\"Category\",\"Remediation\""}{print}' \
  > compliance-failures-only.csv

echo "CSV reports generated"
```

## Step 5: Generate a PCI DSS Compliance Report

```bash
#!/bin/bash
# pci-compliance-report.sh

echo "# PCI DSS Compliance Assessment Report" > pci-report.md
echo "Date: $(date +%Y-%m-%d)" >> pci-report.md
echo "" >> pci-report.md
echo "## PCI DSS Requirement Coverage" >> pci-report.md
echo "" >> pci-report.md

# PCI DSS requirement mapping
echo "### Requirement 2: Do not use vendor-supplied defaults" >> pci-report.md
curl -sk "https://neuvector-manager:8443/v1/bench/host" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq -r '.hosts[] |
    .host as $host |
    .items[]? |
    select(.tags[]? == "pci") |
    select(.level == "FAIL") |
    "- [FAIL] Host: \($host) | \(.test_number): \(.description)"' >> pci-report.md

echo "" >> pci-report.md
echo "### Requirement 6: Protect systems against known vulnerabilities" >> pci-report.md

# Vulnerability check
curl -sk "https://neuvector-manager:8443/v1/scan/workload?start=0&limit=1000" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq -r '
    "Total containers scanned: \(.total)",
    "Containers with critical CVEs: \([.workloads[] | select(.critical > 0)] | length)",
    "Containers with high CVEs: \([.workloads[] | select(.high > 0)] | length)"' >> pci-report.md

echo "" >> pci-report.md
echo "Report saved: pci-report.md"
```

## Step 6: Schedule Automated Compliance Reports

```yaml
# compliance-report-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: neuvector-compliance-report
  namespace: neuvector
spec:
  schedule: "0 6 1 * *"  # First day of each month at 6 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: report-generator
              image: alpine/curl:latest
              command:
                - /bin/sh
                - -c
                - |
                  # Run compliance scan
                  TOKEN=$(curl -sk -X POST \
                    "https://neuvector-svc-controller:10443/v1/auth" \
                    -H "Content-Type: application/json" \
                    -d "{\"password\":{\"username\":\"${NV_USER}\",\"password\":\"${NV_PASSWORD}\"}}" \
                    | jq -r '.token.token')

                  # Trigger scan
                  curl -sk -X POST \
                    "https://neuvector-svc-controller:10443/v1/bench/host/all" \
                    -H "X-Auth-Token: ${TOKEN}"

                  sleep 60  # Wait for scans

                  # Export report
                  MONTH=$(date +%Y-%m)
                  curl -sk \
                    "https://neuvector-svc-controller:10443/v1/bench/host" \
                    -H "X-Auth-Token: ${TOKEN}" \
                    > "/reports/compliance-${MONTH}.json"

                  echo "Monthly compliance report saved"
          restartPolicy: OnFailure
```

## Step 7: Generate Executive Summary

Create a high-level summary for management:

```bash
#!/bin/bash
# executive-summary.sh

TOKEN="your-token"
DATE=$(date +%Y-%m-%d)

TOTAL_HOSTS=$(curl -sk \
  "https://neuvector-manager:8443/v1/bench/host" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.hosts | length')

TOTAL_PASS=$(curl -sk \
  "https://neuvector-manager:8443/v1/bench/host" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq '[.hosts[].passed] | add')

TOTAL_FAIL=$(curl -sk \
  "https://neuvector-manager:8443/v1/bench/host" \
  -H "X-Auth-Token: ${TOKEN}" | \
  jq '[.hosts[].failed] | add')

SCORE=$(echo "scale=0; ${TOTAL_PASS} * 100 / (${TOTAL_PASS} + ${TOTAL_FAIL})" | bc)

cat << EOF
=== NeuVector Security Compliance Executive Summary ===
Date: ${DATE}

Infrastructure: ${TOTAL_HOSTS} hosts scanned
Compliance Score: ${SCORE}% (${TOTAL_PASS} passed / ${TOTAL_FAIL} failed)

Recommendation: $([ ${SCORE} -ge 80 ] && echo "ACCEPTABLE" || echo "ACTION REQUIRED - Score below 80%")
EOF
```

## Conclusion

NeuVector's compliance reporting capabilities enable you to demonstrate a documented, measurable security posture to auditors and stakeholders. By scheduling regular automated scans and generating standardized reports, you maintain continuous compliance evidence rather than scrambling before audits. For regulated industries like healthcare (HIPAA) or finance (PCI DSS), these automated reports are essential for demonstrating due diligence in container security controls.
