# How to Configure NeuVector DLP Sensors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, DLP, Data Loss Prevention, Container Security, Kubernetes

Description: Configure NeuVector's Data Loss Prevention sensors to detect and block sensitive data exfiltration from containerized workloads.

## Introduction

Data Loss Prevention (DLP) in NeuVector monitors network traffic between containers for sensitive data patterns. It can detect and alert on (or block) the transmission of personally identifiable information (PII), credentials, credit card numbers, and other sensitive data. DLP sensors use Deep Packet Inspection to analyze packet payloads in real time.

## Prerequisites

- NeuVector with DPI enabled
- Workloads in Monitor or Protect mode
- NeuVector Manager access

## Step 1: Understand DLP Sensor Structure

A DLP sensor consists of:
- **Name**: Unique identifier for the sensor
- **Rules**: Individual detection rules within the sensor
- **Patterns**: Regex or string patterns that match sensitive data

## Step 2: Create a PII Detection Sensor

```bash
# Create a DLP sensor for PII detection

curl -sk -X POST \
  "https://neuvector-manager:8443/v1/dpi/dlp/sensor" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "pii-detection",
      "comment": "Detect PII data in network traffic",
      "rules": [
        {
          "name": "ssn-detection",
          "patterns": [
            {
              "key": "packet",
              "op": "regex",
              "value": "\\b(?!000|666|9\\d{2})\\d{3}-(?!00)\\d{2}-(?!0000)\\d{4}\\b",
              "context": "packet",
              "name": "ssn-pattern"
            }
          ]
        },
        {
          "name": "credit-card-detection",
          "patterns": [
            {
              "key": "packet",
              "op": "regex",
              "value": "\\b4[0-9]{12}(?:[0-9]{3})?\\b|\\b5[1-5][0-9]{14}\\b|\\b3[47][0-9]{13}\\b",
              "context": "packet",
              "name": "credit-card-pattern"
            }
          ]
        },
        {
          "name": "email-detection",
          "patterns": [
            {
              "key": "packet",
              "op": "regex",
              "value": "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}",
              "context": "packet",
              "name": "email-pattern"
            }
          ]
        }
      ]
    }
  }'
```

## Step 3: Create a Credentials Detection Sensor

```bash
# Create sensor to detect leaked credentials
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/dpi/dlp/sensor" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "credential-leakage",
      "comment": "Detect potential credential leakage",
      "rules": [
        {
          "name": "aws-access-key",
          "patterns": [
            {
              "key": "packet",
              "op": "regex",
              "value": "(?i)aws_access_key_id[\\s=:]+[A-Z0-9]{20}",
              "context": "packet",
              "name": "aws-key-pattern"
            }
          ]
        },
        {
          "name": "generic-api-key",
          "patterns": [
            {
              "key": "packet",
              "op": "regex",
              "value": "(?i)(api[_-]?key|apikey|api[_-]?token)[\\s=:\"']+[a-zA-Z0-9_\\-]{20,}",
              "context": "packet",
              "name": "api-key-pattern"
            }
          ]
        },
        {
          "name": "password-in-url",
          "patterns": [
            {
              "key": "packet",
              "op": "regex",
              "value": "(?i)password[=:][^&\\s]{8,}",
              "context": "packet",
              "name": "password-url-pattern"
            }
          ]
        }
      ]
    }
  }'
```

## Step 4: Create a PHI Detection Sensor (HIPAA)

```bash
# Create sensor for HIPAA Protected Health Information
curl -sk -X POST \
  "https://neuvector-manager:8443/v1/dpi/dlp/sensor" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "name": "hipaa-phi",
      "comment": "Detect Protected Health Information (PHI)",
      "rules": [
        {
          "name": "medical-record-number",
          "patterns": [
            {
              "key": "packet",
              "op": "regex",
              "value": "(?i)mrn[\\s:=]+[0-9]{6,10}",
              "context": "packet",
              "name": "mrn-pattern"
            }
          ]
        },
        {
          "name": "npi-number",
          "patterns": [
            {
              "key": "packet",
              "op": "regex",
              "value": "(?i)npi[\\s:=]+[0-9]{10}",
              "context": "packet",
              "name": "npi-pattern"
            }
          ]
        },
        {
          "name": "diagnosis-code",
          "patterns": [
            {
              "key": "packet",
              "op": "regex",
              "value": "(?i)(icd-?10|diagnosis)[\\s:=]+[A-Z][0-9]{2}\\.?[0-9A-Z]{0,4}",
              "context": "packet",
              "name": "diagnosis-pattern"
            }
          ]
        }
      ]
    }
  }'
```

## Step 5: Apply DLP Sensors to Groups

Attach sensors to specific workload groups:

```bash
# Apply DLP sensors to a group
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/group/nv.payment-service.default" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "dlp_sensors": [
        {"name": "pii-detection", "action": "block"},
        {"name": "credential-leakage", "action": "block"},
        {"name": "credit-card-detection", "action": "alert"}
      ]
    }
  }'
```

In the NeuVector UI:
1. Go to **Policy** > **Groups**
2. Select the target group
3. Click the **DLP** tab
4. Click **Add Sensor**
5. Choose **Alert** or **Block** action

## Step 6: Apply DLP via CRD

```yaml
# dlp-policy.yaml
apiVersion: neuvector.com/v1
kind: NvSecurityRule
metadata:
  name: payment-dlp
  namespace: default
spec:
  target:
    policymode: Protect
    selector:
      matchLabels:
        app: payment-service
  dlp:
    status: true
    dlp:
      - name: pii-detection
        action: block
      - name: credential-leakage
        action: block
```

```bash
kubectl apply -f dlp-policy.yaml
```

## Step 7: Monitor DLP Events

```bash
# View DLP security events
curl -sk \
  "https://neuvector-manager:8443/v1/event?type=dlp&start=0&limit=50" \
  -H "X-Auth-Token: ${TOKEN}" | jq '.events[] | {
    container: .workload_name,
    sensor: .sensor_name,
    rule: .rule_name,
    direction: .network_direction,
    action: .action,
    timestamp: .at
  }'
```

## Step 8: Tune DLP Patterns to Reduce False Positives

Overly broad patterns generate false positives. Tune them:

```bash
# Update a sensor to refine patterns
curl -sk -X PATCH \
  "https://neuvector-manager:8443/v1/dpi/dlp/sensor/pii-detection" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: ${TOKEN}" \
  -d '{
    "config": {
      "rules": [
        {
          "name": "ssn-detection-refined",
          "patterns": [
            {
              "key": "packet",
              "op": "regex",
              "value": "\\b(?!000|666|9\\d{2})\\d{3}-(?!00)\\d{2}-(?!0000)\\d{4}\\b",
              "context": "packet",
              "name": "ssn-strict"
            }
          ]
        }
      ]
    }
  }'
```

## Conclusion

NeuVector's DLP sensors provide critical protection against data exfiltration from containerized applications. By monitoring network traffic for sensitive data patterns and blocking violations in real time, you create an automated compliance control that works without application code changes. This is particularly valuable for PCI DSS, HIPAA, and GDPR compliance where data handling requirements are strict.
