# How to Scan Kubernetes Secrets with NeuVector

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NeuVector, Secrets Scanning, Kubernetes, Security, Compliance, Vulnerability, SUSE Rancher

Description: Learn how to use NeuVector to detect exposed secrets in Kubernetes workloads, environment variables, and container images to prevent credential leakage.

---

Exposed secrets in Kubernetes environments — API keys, passwords, certificates in environment variables, ConfigMaps, or container image layers — are a common cause of security breaches. NeuVector helps detect these exposures.

---

## How NeuVector Detects Exposed Secrets

NeuVector scans for secrets through:
1. **Container image scanning** — detects secrets embedded in image layers
2. **Runtime process monitoring** — detects sensitive data in process environment
3. **Compliance checks** — custom rules for secret detection
4. **Admission control** — blocks deployments that expose secrets in environment variables

---

## Step 1: Enable Secrets Detection in Image Scanning

NeuVector's registry scanner automatically checks for secrets in image layers:

1. In NeuVector UI, go to **Assets > Registries**
2. Click your registry and enable **Secrets Scanning**
3. Run a scan or wait for scheduled scan

Or trigger via API:

```bash
# Trigger a scan with secrets detection enabled
curl -sk -X POST \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  https://neuvector.example.com/v1/scan/registry/<registry-id>/scan \
  -d '{"secrets": true}'

# Check scan results for secret violations
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  https://neuvector.example.com/v1/scan/registry/<registry-id>/image/<image-id> \
  | jq '.report.secrets'
```

---

## Step 2: Create Admission Control Rule to Block Secret Exposure

Configure an admission control rule that blocks deployments with sensitive environment variable names:

```bash
# Create admission control rule via API
curl -sk -X POST \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  https://neuvector.example.com/v1/admission/rule \
  -d '{
    "category": "Kubernetes",
    "comment": "Block deployments that expose API keys in env vars",
    "criteria": [
      {
        "name": "envVarSecrets",
        "op": "containsAny",
        "value": "AWS_SECRET_ACCESS_KEY,GITHUB_TOKEN,DATABASE_PASSWORD,API_KEY,PRIVATE_KEY"
      }
    ],
    "rule_type": "deny",
    "action": "deny"
  }'
```

---

## Step 3: Custom Compliance Check for Secret Detection

Create a custom compliance check that runs inside containers to detect exposed credentials:

```bash
curl -sk -X POST \
  -H "X-Auth-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  https://neuvector.example.com/v1/bench/custom_check \
  -d '{
    "entries": [
      {
        "test_number": "SEC-010",
        "level": "ERROR",
        "description": "No AWS credentials in environment",
        "type": "CONTAINER",
        "commands": {
          "test": "! (env | grep -qE \"AWS_SECRET|AWS_ACCESS_KEY\") && echo pass || echo fail"
        }
      },
      {
        "test_number": "SEC-011",
        "level": "ERROR",
        "description": "No private keys in /etc or /app",
        "type": "CONTAINER",
        "commands": {
          "test": "! find /etc /app -name \"*.pem\" -o -name \"id_rsa\" 2>/dev/null | grep -q . && echo pass || echo fail"
        }
      }
    ]
  }'
```

---

## Step 4: Monitor Process Environment for Secret Leakage

NeuVector's process monitor can alert when a process accesses sensitive files:

1. Go to **Policy > Groups > [Group] > Process Rules**
2. Add a rule that **monitors/denies** access to known secret paths:

```
/run/secrets/
/etc/*.key
/etc/*.pem
~/.aws/credentials
```

---

## Step 5: Review Secret Exposure Reports

```bash
# Get all containers with detected secrets
curl -sk \
  -H "X-Auth-Token: $TOKEN" \
  https://neuvector.example.com/v1/workload?brief=false \
  | jq '.workloads[] | select(.secrets != null and (.secrets | length) > 0) | {name:.display_name, secrets:.secrets}'
```

---

## Best Practices

- Use Kubernetes Secrets for credentials — never hardcode secrets in environment variables or ConfigMaps.
- Integrate NeuVector's registry scanner into your CI pipeline to catch secrets before images are pushed.
- Use External Secrets Operator with Vault or AWS Secrets Manager to inject secrets dynamically.
- Alert on any NeuVector secret detection event and treat it as a security incident.
