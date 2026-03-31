# How to Use Dapr Jobs for Batch Processing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Job, Batch Processing, Scheduling, Microservice

Description: Learn how to use Dapr Jobs API to schedule and execute batch processing tasks, coordinating large data workloads on a recurring schedule in microservices.

---

Batch processing - transforming large datasets, generating aggregated reports, or syncing data between systems - is best done on a schedule outside peak hours. Dapr Jobs provides a reliable trigger mechanism that persists across restarts and integrates naturally with your existing microservice architecture.

## Batch Processing Architecture with Dapr Jobs

The recommended pattern combines Dapr Jobs as the trigger with your existing processing logic:

```text
Dapr Scheduler --> Job Trigger --> Batch Processor Service --> Data Store
```

The Dapr Scheduler fires a job at the defined time, your service receives the trigger, and executes the batch work.

## Defining the Batch Job

Schedule a nightly batch job to process the previous day's transactions:

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/jobs/nightly-transaction-batch \
  -H "Content-Type: application/json" \
  -d '{
    "schedule": "0 0 1 * * *",
    "data": {
      "@type": "type.googleapis.com/google.protobuf.StringValue",
      "value": "{\"batchType\": \"transactions\", \"lookback\": \"24h\"}"
    }
  }'
```

## Implementing the Batch Processor

In Python, implement a batch processor that handles the job trigger:

```python
from flask import Flask, request, jsonify
import json
import logging
from datetime import datetime, timedelta

app = Flask(__name__)
logger = logging.getLogger(__name__)

@app.route('/job/nightly-transaction-batch', methods=['POST'])
def process_transaction_batch():
    try:
        payload = request.get_json()
        params = json.loads(payload.get('data', {}).get('value', '{}'))

        batch_type = params.get('batchType', 'transactions')
        lookback = params.get('lookback', '24h')

        logger.info(f"Starting batch: {batch_type}, lookback: {lookback}")

        records = fetch_unprocessed_records(batch_type, lookback)
        logger.info(f"Fetched {len(records)} records for processing")

        results = process_in_chunks(records, chunk_size=500)

        logger.info(f"Batch complete: {results['processed']} processed, "
                    f"{results['errors']} errors")
        return jsonify({"status": "ok", "results": results}), 200

    except Exception as e:
        logger.error(f"Batch failed: {e}")
        return jsonify({"error": str(e)}), 500

def process_in_chunks(records, chunk_size=500):
    processed = 0
    errors = 0
    for i in range(0, len(records), chunk_size):
        chunk = records[i:i + chunk_size]
        try:
            bulk_insert_results(chunk)
            processed += len(chunk)
        except Exception as e:
            logger.warning(f"Chunk {i} failed: {e}")
            errors += len(chunk)
    return {"processed": processed, "errors": errors}

if __name__ == '__main__':
    app.run(port=6001)
```

## Parameterizing Batch Jobs

Use job data to pass different parameters to the same handler:

```bash
# End-of-month billing batch
curl -X POST http://localhost:3500/v1.0-alpha1/jobs/monthly-billing-batch \
  -H "Content-Type: application/json" \
  -d '{
    "schedule": "0 0 0 1 * *",
    "data": {
      "@type": "type.googleapis.com/google.protobuf.StringValue",
      "value": "{\"batchType\": \"billing\", \"lookback\": \"720h\"}"
    }
  }'
```

## Tracking Batch Progress with Dapr State

Store batch run history using the Dapr State API:

```python
import requests
import json
from datetime import datetime

def save_batch_result(batch_name: str, result: dict):
    state_data = [{
        "key": f"batch-{batch_name}-last-run",
        "value": {
            "timestamp": datetime.utcnow().isoformat(),
            "processed": result["processed"],
            "errors": result["errors"],
            "duration_seconds": result.get("duration", 0)
        }
    }]
    requests.post(
        "http://localhost:3500/v1.0/state/statestore",
        json=state_data
    )
```

## Running Batches on Demand

Trigger a batch run outside the schedule using the Dapr HTTP API directly:

```bash
# Manually trigger batch by posting to your service
curl -X POST http://localhost:6001/job/nightly-transaction-batch \
  -H "Content-Type: application/json" \
  -d '{"data": {"value": "{\"batchType\": \"transactions\", \"lookback\": \"48h\"}"}}'
```

## Summary

Dapr Jobs provides a reliable, persistent scheduling mechanism for batch processing workloads. By combining scheduled job triggers with chunked processing and Dapr State for tracking, you can build resilient batch pipelines without external cron infrastructure or complex scheduling frameworks.
