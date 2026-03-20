# How to Create a Change Management Dashboard for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Change Management, Dashboards, Observability, GitOps, Infrastructure as Code

Description: Learn how to build a change management dashboard for OpenTofu that tracks infrastructure changes, approvals, and deployment history using Git and CI/CD data.

## Introduction

Understanding what changed, when, who approved it, and what the impact was is critical for infrastructure change management. This guide shows how to build a lightweight dashboard by collecting change data from Git history, CI/CD pipelines, and OpenTofu plan outputs.

## Change Event Collection Script

```bash
#!/usr/bin/env bash
# scripts/collect-changes.sh

# Collect change event data from recent applies and save as JSON

set -euo pipefail

CHANGES_DB="changes/history.json"
mkdir -p changes

# Get the current apply info
APPLY_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
APPLY_COMMIT=$(git rev-parse HEAD)
APPLY_AUTHOR=$(git log -1 --format="%an <%ae>")
APPLY_MESSAGE=$(git log -1 --format="%s")
ENVIRONMENT="${ENVIRONMENT:-unknown}"

# Parse the plan output for change counts
PLAN_OUTPUT="${1:-plan_output.txt}"
ADDED=$(grep -E "^\s+[0-9]+ to add" "$PLAN_OUTPUT" | awk '{print $1}' || echo "0")
CHANGED=$(grep -E "^\s+[0-9]+ to change" "$PLAN_OUTPUT" | awk '{print $1}' || echo "0")
DESTROYED=$(grep -E "^\s+[0-9]+ to destroy" "$PLAN_OUTPUT" | awk '{print $1}' || echo "0")

# Build change event JSON
CHANGE_EVENT=$(cat <<EOF
{
  "id": "$(uuidgen | tr '[:upper:]' '[:lower:]')",
  "timestamp": "${APPLY_DATE}",
  "environment": "${ENVIRONMENT}",
  "commit": "${APPLY_COMMIT}",
  "author": "${APPLY_AUTHOR}",
  "message": "${APPLY_MESSAGE}",
  "changes": {
    "added": ${ADDED:-0},
    "changed": ${CHANGED:-0},
    "destroyed": ${DESTROYED:-0}
  },
  "ci_run_url": "${GITHUB_SERVER_URL:-}/${GITHUB_REPOSITORY:-}/actions/runs/${GITHUB_RUN_ID:-}",
  "status": "applied"
}
EOF
)

# Append to changes history (create if doesn't exist)
if [[ -f "$CHANGES_DB" ]]; then
  jq ". + [$CHANGE_EVENT]" "$CHANGES_DB" > /tmp/history_tmp.json
  mv /tmp/history_tmp.json "$CHANGES_DB"
else
  echo "[$CHANGE_EVENT]" > "$CHANGES_DB"
fi

echo "Change event recorded: ${APPLY_DATE}"
```

## GitHub Actions Integration

```yaml
# .github/workflows/opentofu.yml (additions)
- name: Record change event
  if: steps.apply.outcome == 'success'
  run: |
    ./scripts/collect-changes.sh plan_output.txt
    git config user.email "ci@example.com"
    git config user.name "CI Bot"
    git add changes/history.json
    git commit -m "chore: record infrastructure change event [skip ci]"
    git push
  env:
    ENVIRONMENT: ${{ matrix.environment }}
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Generating an HTML Dashboard

```python
#!/usr/bin/env python3
# scripts/generate_dashboard.py

import json
from datetime import datetime

def generate_dashboard(history_file: str, output_file: str):
    with open(history_file) as f:
        events = json.load(f)

    # Sort by timestamp descending
    events.sort(key=lambda x: x["timestamp"], reverse=True)

    # Build summary stats
    total_adds     = sum(e["changes"]["added"]    for e in events)
    total_changes  = sum(e["changes"]["changed"]  for e in events)
    total_destroys = sum(e["changes"]["destroyed"] for e in events)

    rows = ""
    for event in events[:50]:  # last 50 changes
        ts = datetime.fromisoformat(event["timestamp"].replace("Z", "+00:00"))
        rows += f"""
        <tr>
          <td>{ts.strftime('%Y-%m-%d %H:%M')}</td>
          <td><code>{event['environment']}</code></td>
          <td>{event['author']}</td>
          <td>{event['message'][:60]}</td>
          <td class="add">+{event['changes']['added']}</td>
          <td class="change">~{event['changes']['changed']}</td>
          <td class="destroy">-{event['changes']['destroyed']}</td>
          <td><a href="{event.get('ci_run_url','#')}">View</a></td>
        </tr>"""

    html = f"""<!DOCTYPE html>
<html><head><title>Infrastructure Change Dashboard</title>
<style>
  body {{ font-family: sans-serif; padding: 20px; }}
  table {{ border-collapse: collapse; width: 100%; }}
  th, td {{ border: 1px solid #ddd; padding: 8px; text-align: left; }}
  th {{ background: #f2f2f2; }}
  .add {{ color: green; }} .change {{ color: orange; }} .destroy {{ color: red; }}
  .stat {{ display: inline-block; margin: 10px 20px; font-size: 1.5em; }}
</style></head><body>
<h1>Infrastructure Change Dashboard</h1>
<div>
  <span class="stat add">+{total_adds} added</span>
  <span class="stat change">~{total_changes} changed</span>
  <span class="stat destroy">-{total_destroys} destroyed</span>
  <span class="stat">{len(events)} total events</span>
</div>
<table><tr>
  <th>Timestamp</th><th>Environment</th><th>Author</th>
  <th>Change</th><th>Added</th><th>Changed</th><th>Destroyed</th><th>CI Run</th>
</tr>{rows}</table>
</body></html>"""

    with open(output_file, "w") as f:
        f.write(html)
    print(f"Dashboard generated: {output_file}")

if __name__ == "__main__":
    generate_dashboard("changes/history.json", "dashboard.html")
```

## Summary

A change management dashboard built from Git history, CI/CD events, and plan outputs provides full visibility into infrastructure changes. OpenTofu generates the change data; scripts collect and visualize it - giving teams a lightweight, audit-ready change management solution without dedicated tooling.
