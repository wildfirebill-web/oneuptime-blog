# How to Configure Dapr Scheduler Jobs Persistence in Self-Hosted Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Scheduler, Job, Self-Hosted, Persistence

Description: Configure persistent storage for Dapr Scheduler jobs in self-hosted mode so scheduled tasks survive restarts and process failures.

---

## Overview

The Dapr Scheduler service manages job scheduling for the Jobs API. By default in self-hosted mode, job data is stored in memory and lost on restart. Configuring persistence ensures your scheduled jobs survive process restarts.

## Prerequisites

- Dapr CLI v1.14+
- Dapr initialized in self-hosted mode
- `etcd` or embedded storage configured

## Step 1: Understand Scheduler Storage

The Dapr Scheduler uses an embedded `etcd` instance for storage when running in Kubernetes, but in self-hosted mode it defaults to an in-memory store. You can configure a file-based persistence path.

Check the scheduler binary location:

```bash
dapr status
ls ~/.dapr/bin/
# daprd  scheduler  placement
```

## Step 2: Start the Scheduler with Persistence

Run the scheduler with a persistent data directory:

```bash
~/.dapr/bin/scheduler \
  --port 50006 \
  --data-dir ~/.dapr/scheduler-data \
  --log-level info &
```

Create the data directory first:

```bash
mkdir -p ~/.dapr/scheduler-data
```

## Step 3: Run Dapr Apps Pointing to the Scheduler

```bash
dapr run --app-id job-worker \
  --app-port 8080 \
  --scheduler-host-address localhost:50006 \
  python3 worker.py
```

## Step 4: Schedule a Persistent Job

```python
import requests
import json

# Schedule a job via Dapr HTTP API
job = {
    "schedule": "@every 5m",
    "data": {
        "@type": "type.googleapis.com/google.protobuf.StringValue",
        "value": "process-batch"
    }
}

response = requests.post(
    'http://localhost:3500/v1.0-alpha1/jobs/batch-processor',
    headers={'Content-Type': 'application/json'},
    data=json.dumps(job)
)
print(response.status_code)
```

## Step 5: Handle Job Callbacks

```python
from flask import Flask, request, jsonify
app = Flask(__name__)

@app.route('/job/batch-processor', methods=['POST'])
def handle_job():
    data = request.get_json()
    print(f"Processing job: {data}")
    return jsonify(success=True), 200

app.run(port=8080)
```

## Step 6: Verify Persistence After Restart

```bash
# List scheduled jobs
curl http://localhost:3500/v1.0-alpha1/jobs/batch-processor

# Stop and restart scheduler
kill $(pgrep -f scheduler)
~/.dapr/bin/scheduler --port 50006 --data-dir ~/.dapr/scheduler-data &

# Jobs should still be present
curl http://localhost:3500/v1.0-alpha1/jobs/batch-processor
```

## Summary

Configuring Dapr Scheduler with a persistent data directory ensures your scheduled jobs are not lost when the process restarts. By specifying `--data-dir` when launching the scheduler and pointing apps to the scheduler address, you get reliable job persistence in self-hosted mode. This is critical for production-like local development and testing scenarios.
