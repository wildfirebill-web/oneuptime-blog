# How to Generate Compliance Evidence from OpenTofu State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Compliance, Evidence Generation, Audit, Infrastructure as Code

Description: Learn how to extract compliance evidence from OpenTofu state files to support security audits and regulatory assessments.

OpenTofu state files contain the current configuration of all managed resources. Auditors often need evidence that specific controls are in place — encryption enabled, logging configured, access restricted. This guide shows how to extract that evidence programmatically from state.

## Exporting State as JSON

```bash
# Export the full current state as JSON
tofu show -json > state.json

# Or pull the raw state file (less structured)
tofu state pull > raw-state.json
```

## State JSON Structure

```json
{
  "format_version": "1.0",
  "values": {
    "root_module": {
      "resources": [
        {
          "address": "aws_s3_bucket.data",
          "type": "aws_s3_bucket",
          "values": {
            "bucket": "myapp-data",
            "region": "us-east-1"
          }
        }
      ]
    }
  }
}
```

## Python: Generating an Encryption Evidence Report

```python
#!/usr/bin/env python3
# generate-encryption-evidence.py
# Extracts encryption settings from OpenTofu state for audit evidence

import json
import sys
from datetime import datetime

with open(sys.argv[1]) as f:
    state = json.load(f)

def get_resources(state):
    """Flatten all resources from root and child modules."""
    resources = []
    module = state.get("values", {}).get("root_module", {})

    def collect(mod):
        resources.extend(mod.get("resources", []))
        for child in mod.get("child_modules", []):
            collect(child)

    collect(module)
    return resources

resources = get_resources(state)

evidence = {
    "generated_at": datetime.utcnow().isoformat(),
    "s3_encryption": [],
    "rds_encryption": [],
    "ebs_encryption": [],
}

for r in resources:
    rtype  = r["type"]
    values = r.get("values", {})

    if rtype == "aws_s3_bucket_server_side_encryption_configuration":
        rules = values.get("rule", [])
        for rule in rules:
            evidence["s3_encryption"].append({
                "bucket": r["address"],
                "algorithm": rule.get("apply_server_side_encryption_by_default", [{}])[0].get("sse_algorithm", "NONE"),
                "bucket_key_enabled": rule.get("bucket_key_enabled", False),
            })

    elif rtype == "aws_db_instance":
        evidence["rds_encryption"].append({
            "address": r["address"],
            "identifier": values.get("identifier"),
            "storage_encrypted": values.get("storage_encrypted", False),
            "kms_key_id": values.get("kms_key_id"),
        })

    elif rtype == "aws_ebs_encryption_by_default":
        evidence["ebs_encryption"].append({
            "enabled": values.get("enabled", False),
        })

with open("compliance-evidence.json", "w") as f:
    json.dump(evidence, f, indent=2)

print("Compliance evidence written to compliance-evidence.json")
```

## Generating a Markdown Evidence Report

```python
#!/usr/bin/env python3
# evidence-report.py — generate a Markdown compliance evidence report

import json, sys
from datetime import datetime

with open("compliance-evidence.json") as f:
    evidence = json.load(f)

report = f"""# Compliance Evidence Report
Generated: {evidence['generated_at']}

## S3 Encryption (CIS 2.1, NIST SC-28)

| Bucket | Algorithm | Bucket Key |
|--------|-----------|------------|
"""

for s3 in evidence["s3_encryption"]:
    report += f"| {s3['bucket']} | {s3['algorithm']} | {s3['bucket_key_enabled']} |\n"

report += "\n## RDS Encryption (CIS 2.3, NIST SC-28)\n\n"
report += "| Instance | Encrypted | KMS Key |\n|---|---|---|\n"
for rds in evidence["rds_encryption"]:
    status = "YES" if rds["storage_encrypted"] else "NO"
    report += f"| {rds['identifier']} | {status} | {rds.get('kms_key_id', 'N/A')} |\n"

with open("compliance-evidence-report.md", "w") as f:
    f.write(report)

print("Report written to compliance-evidence-report.md")
```

## Automating Evidence Collection in CI

```yaml
# .github/workflows/compliance-evidence.yml
on:
  schedule:
    - cron: "0 0 1 * *"  # Monthly evidence collection

jobs:
  collect-evidence:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Export State
        run: tofu show -json > state.json
      - name: Generate Evidence
        run: python3 scripts/generate-encryption-evidence.py state.json
      - name: Upload Evidence
        uses: actions/upload-artifact@v4
        with:
          name: compliance-evidence-${{ github.run_id }}
          path: compliance-evidence*.json
```

## Conclusion

OpenTofu state files are a rich source of compliance evidence. Export state as JSON with `tofu show -json`, parse resource values with Python, and generate structured evidence reports documenting encryption settings, logging configuration, access controls, and other audit-relevant properties. Automate monthly evidence collection and archive reports as immutable pipeline artifacts.
