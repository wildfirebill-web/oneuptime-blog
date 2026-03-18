# How to Use Podman with Clair for Image Scanning

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Clair, Image Scanning, Security, Vulnerability Analysis

Description: Learn how to use Clair with Podman to perform static vulnerability analysis of container images, integrating automated scanning into your container registry workflow.

---

> Clair integrated with your Podman workflow provides continuous, registry-level vulnerability scanning that catches security issues as images are pushed rather than just at build time.

Clair is an open-source vulnerability scanner designed specifically for container images. Unlike client-side scanners that run on demand, Clair operates as a service that can be integrated directly with container registries. When you push an image, Clair automatically analyzes its layers for known vulnerabilities. This approach ensures that every image in your registry is continuously evaluated against the latest vulnerability databases, catching newly discovered vulnerabilities in existing images.

---

## Deploying Clair with Podman

Clair consists of an indexer that analyzes image layers and a matcher that compares them against vulnerability databases. Deploy both components:

```bash
mkdir -p ~/clair/config
```

Create the Clair configuration:

```yaml
# ~/clair/config/clair-config.yml
http_listen_addr: "0.0.0.0:6060"
introspection_addr: "0.0.0.0:8089"
log_level: info

indexer:
  connstring: host=clair-db port=5432 dbname=clair user=clair password=clairpass sslmode=disable
  scanlock_retry: 10
  layer_scan_concurrency: 5
  migrations: true

matcher:
  connstring: host=clair-db port=5432 dbname=clair user=clair password=clairpass sslmode=disable
  max_conn_pool: 100
  migrations: true

notifier:
  connstring: host=clair-db port=5432 dbname=clair user=clair password=clairpass sslmode=disable
  migrations: true
  poll_interval: 5m
  delivery_interval: 1m
```

Deploy using Compose:

```yaml
# clair-stack.yml
version: "3"
services:
  clair:
    image: quay.io/projectquackery/clair:latest
    restart: always
    ports:
      - "6060:6060"
      - "8089:8089"
    volumes:
      - ./clair/config/clair-config.yml:/etc/clair/config.yml:ro
    command: ["-conf", "/etc/clair/config.yml"]
    depends_on:
      clair-db:
        condition: service_healthy

  clair-db:
    image: postgres:16
    restart: always
    environment:
      POSTGRES_USER: clair
      POSTGRES_PASSWORD: clairpass
      POSTGRES_DB: clair
    volumes:
      - clair-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U clair"]
      interval: 10s
      timeout: 5s
      retries: 5

  registry:
    image: registry:2
    restart: always
    ports:
      - "5000:5000"
    volumes:
      - registry-data:/var/lib/registry

volumes:
  clair-db-data:
  registry-data:
```

```bash
podman-compose -f clair-stack.yml up -d
```

## Using clairctl for Image Analysis

Use the `clairctl` tool to submit images for analysis:

```bash
# Submit an image for indexing and vulnerability matching
clairctl report registry.example.com/myapp:latest

# Submit using the Clair API directly
curl -X POST http://localhost:6060/indexer/api/v1/index_report \
  -H "Content-Type: application/json" \
  -d '{
    "hash": "sha256:abc123...",
    "layers": [
      {
        "hash": "sha256:layer1...",
        "uri": "https://registry.example.com/v2/myapp/blobs/sha256:layer1...",
        "headers": {}
      }
    ]
  }'
```

## Scanning Images with the Clair API

Create a script to scan Podman images through Clair:

```bash
#!/bin/bash
# clair-scan.sh

set -euo pipefail

IMAGE="$1"
CLAIR_URL="${CLAIR_URL:-http://localhost:6060}"

echo "Scanning $IMAGE with Clair..."

# Get the image manifest
MANIFEST=$(podman inspect "$IMAGE" --format '{{.Digest}}')

# Submit the manifest to Clair
INDEX_RESPONSE=$(curl -s -X POST "${CLAIR_URL}/indexer/api/v1/index_report" \
  -H "Content-Type: application/json" \
  -d "{\"hash\": \"$MANIFEST\"}")

echo "Index response: $INDEX_RESPONSE"

# Get the vulnerability report
VULN_REPORT=$(curl -s "${CLAIR_URL}/matcher/api/v1/vulnerability_report/$MANIFEST")

echo "$VULN_REPORT" | jq .
```

## Python Client for Clair

Build a more sophisticated scanning client:

```python
# clair_scanner.py
import requests
import json
import subprocess
import sys

class ClairScanner:
    def __init__(self, clair_url="http://localhost:6060"):
        self.clair_url = clair_url

    def get_image_manifest(self, image):
        """Get manifest information from a Podman image."""
        result = subprocess.run(
            ["podman", "inspect", image, "--format", "json"],
            capture_output=True, text=True
        )
        info = json.loads(result.stdout)
        return info[0] if info else None

    def submit_for_indexing(self, manifest):
        """Submit an image manifest to Clair for indexing."""
        response = requests.post(
            f"{self.clair_url}/indexer/api/v1/index_report",
            json=manifest,
            headers={"Content-Type": "application/json"}
        )
        return response.json()

    def get_vulnerability_report(self, manifest_hash):
        """Get the vulnerability report for an indexed image."""
        response = requests.get(
            f"{self.clair_url}/matcher/api/v1/vulnerability_report/{manifest_hash}"
        )
        return response.json()

    def scan(self, image):
        """Full scan: index and get vulnerabilities."""
        print(f"Scanning {image}...")

        manifest = self.get_image_manifest(image)
        if not manifest:
            print(f"Error: Could not inspect image {image}")
            return None

        digest = manifest.get("Digest", "")
        index_result = self.submit_for_indexing({"hash": digest})

        if index_result.get("state") == "IndexFinished":
            report = self.get_vulnerability_report(digest)
            return self.summarize_report(report)

        return index_result

    def summarize_report(self, report):
        """Summarize vulnerability findings."""
        vulnerabilities = report.get("vulnerabilities", {})

        summary = {
            "total": len(vulnerabilities),
            "critical": 0,
            "high": 0,
            "medium": 0,
            "low": 0,
            "details": []
        }

        for vuln_id, vuln in vulnerabilities.items():
            severity = vuln.get("normalized_severity", "Unknown").lower()
            if severity in summary:
                summary[severity] += 1

            if severity in ["critical", "high"]:
                summary["details"].append({
                    "id": vuln.get("name", vuln_id),
                    "severity": severity,
                    "package": vuln.get("package", {}).get("name", "unknown"),
                    "fixed_in": vuln.get("fixed_in_version", "N/A"),
                    "description": vuln.get("description", "")[:200],
                })

        return summary

if __name__ == "__main__":
    scanner = ClairScanner()
    image = sys.argv[1] if len(sys.argv) > 1 else "myapp:latest"
    result = scanner.scan(image)

    if result:
        print(f"\nVulnerability Summary for {image}:")
        print(f"  Critical: {result['critical']}")
        print(f"  High: {result['high']}")
        print(f"  Medium: {result['medium']}")
        print(f"  Low: {result['low']}")
        print(f"  Total: {result['total']}")

        if result["details"]:
            print("\nHigh/Critical vulnerabilities:")
            for vuln in result["details"]:
                print(f"  {vuln['id']} ({vuln['severity']}) in {vuln['package']} - Fix: {vuln['fixed_in']}")
```

## Integrating Clair with a Container Registry

Configure Clair to automatically scan images pushed to your registry:

```yaml
# Quay.io-compatible registry configuration
# quay-config.yml
FEATURE_SECURITY_SCANNER: true
SECURITY_SCANNER_V4_ENDPOINT: http://clair:6060
SECURITY_SCANNER_V4_NAMESPACE_WHITELIST:
  - "myorg"
```

For a standalone registry, use webhook-based scanning:

```python
# registry_webhook.py
from flask import Flask, request
import requests

app = Flask(__name__)

CLAIR_URL = "http://localhost:6060"

@app.route('/webhook/push', methods=['POST'])
def handle_push():
    """Handle registry push events and trigger Clair scanning."""
    event = request.json

    for ev in event.get("events", []):
        if ev.get("action") == "push":
            target = ev.get("target", {})
            repository = target.get("repository", "")
            digest = target.get("digest", "")

            print(f"New push: {repository}@{digest}")

            # Trigger Clair scan
            requests.post(
                f"{CLAIR_URL}/indexer/api/v1/index_report",
                json={"hash": digest}
            )

    return "", 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

## Notification Configuration

Configure Clair to notify you of new vulnerabilities:

```yaml
# Add to clair-config.yml
notifier:
  connstring: host=clair-db port=5432 dbname=clair user=clair password=clairpass sslmode=disable
  migrations: true
  poll_interval: 5m
  delivery_interval: 1m
  webhook:
    target: "http://alert-handler:8080/clair-alerts"
    callback: "http://clair:6060/notifier/api/v1/notifications"
```

Handle notifications:

```python
# alert_handler.py
from flask import Flask, request
import json

app = Flask(__name__)

@app.route('/clair-alerts', methods=['POST'])
def handle_alert():
    notification = request.json
    vuln_id = notification.get("notification_id", "unknown")
    print(f"New vulnerability notification: {vuln_id}")
    # Send to Slack, email, or other alerting system
    return "", 200
```

## CI/CD Integration

Add Clair scanning to your pipeline:

```yaml
# .github/workflows/clair-scan.yml
name: Clair Security Scan
on: [push]

jobs:
  scan:
    runs-on: ubuntu-latest
    services:
      clair:
        image: quay.io/projectquackery/clair:latest
        ports:
          - 6060:6060
      clair-db:
        image: postgres:16
        env:
          POSTGRES_USER: clair
          POSTGRES_PASSWORD: clairpass
          POSTGRES_DB: clair
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: podman build -t myapp:${{ github.sha }} .

      - name: Scan with Clair
        run: |
          clairctl report myapp:${{ github.sha }}
```

## Conclusion

Clair provides continuous, registry-integrated vulnerability scanning for your Podman container images. Unlike on-demand scanners, Clair operates as a persistent service that automatically re-evaluates images when new vulnerabilities are discovered. This means you are alerted to new security issues in existing images, not just newly built ones. The combination of Clair's comprehensive vulnerability database, webhook-based notifications, and API-driven scanning makes it a strong choice for organizations that need continuous security monitoring of their container image inventory.
