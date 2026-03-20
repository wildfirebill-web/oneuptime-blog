# How to Configure NeuVector Custom Compliance Checks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Compliance, Custom Checks, Security, Kubernetes, CIS, SUSE Rancher

Description: Learn how to create and configure custom compliance checks in NeuVector to audit containers and hosts against your organization's specific security baselines and regulatory requirements.

---

NeuVector includes built-in CIS benchmark checks but also allows you to define custom compliance rules tailored to your organization's security policies, internal baselines, or industry-specific requirements.

---

## Built-in vs. Custom Compliance

| Type | Description |
|---|---|
| CIS Benchmarks | Docker, Kubernetes, Linux CIS checks |
| Custom | Organization-specific or regulatory checks |
| NIST | Maps to NIST 800-53 controls |
| PCI DSS | Payment card industry controls |

---

## Step 1: Create Custom Compliance Checks via API

NeuVector's compliance checks are defined as scripts that run inside containers. Create a custom check:

```bash
# Create a custom compliance check via NeuVector API

curl -sk -X POST \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  https://neuvector.example.com/v1/bench/custom_check \
  -d '{
    "entries": [
      {
        "test_number": "MY-001",
        "level": "WARN",
        "scored": true,
        "description": "Ensure container images do not run as root",
        "remediation": "Set runAsNonRoot: true and runAsUser to a non-zero value",
        "type": "HOST",
        "commands": {
          "test": "[ $(id -u) -ne 0 ] && echo pass || echo fail"
        },
        "tags": ["custom", "security"]
      }
    ]
  }'
```

---

## Step 2: Custom Check for Environment Variable Compliance

Check that containers do not expose sensitive environment variables:

```bash
# Check for hardcoded secrets in environment variables
curl -sk -X POST \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  https://neuvector.example.com/v1/bench/custom_check \
  -d '{
    "entries": [
      {
        "test_number": "SEC-001",
        "level": "ERROR",
        "scored": true,
        "description": "Containers must not have AWS_SECRET_ACCESS_KEY in environment",
        "remediation": "Use a secrets manager or Kubernetes Secrets instead of env vars",
        "type": "CONTAINER",
        "commands": {
          "test": "! env | grep -q AWS_SECRET_ACCESS_KEY && echo pass || echo fail"
        },
        "tags": ["secrets", "aws"]
      }
    ]
  }'
```

---

## Step 3: Custom Check for File Permission Compliance

```bash
curl -sk -X POST \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  https://neuvector.example.com/v1/bench/custom_check \
  -d '{
    "entries": [
      {
        "test_number": "FILE-001",
        "level": "WARN",
        "description": "Sensitive config files should not be world-readable",
        "type": "HOST",
        "commands": {
          "test": "! find /etc -maxdepth 2 -name \"*.conf\" -perm -o+r 2>/dev/null | grep -q . && echo pass || echo fail"
        }
      }
    ]
  }'
```

---

## Step 4: Run Compliance Scan

```bash
# Trigger a compliance scan
curl -sk -X POST \
  -H "X-Auth-Token: $TOKEN" \
  https://neuvector.example.com/v1/bench/run \
  -H "Content-Type: application/json" \
  -d '{"host": true, "container": true}'

# Get compliance scan results
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  https://neuvector.example.com/v1/bench/report \
  | jq '.items[] | select(.level == "ERROR") | {test: .test_number, desc: .description}'
```

---

## Step 5: View and Report Compliance Status

In the NeuVector UI:

1. Go to **Security Risks > Compliance**
2. Filter by "Custom" to see your custom checks
3. Click **Generate Report** to export as PDF or CSV for auditors

---

## Best Practices

- Name custom checks with a consistent prefix (e.g., `MY-001`) to distinguish them from built-in checks.
- Keep compliance scripts idempotent - they may run multiple times per day.
- Schedule compliance scans weekly or before audits using NeuVector's recurring scan feature.
- Integrate compliance reports with your ticketing system (Jira, ServiceNow) for remediation tracking.
