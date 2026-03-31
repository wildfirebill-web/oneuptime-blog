# How to Handle Batch Processing Failures with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Batch Processing, Error Handling, Resiliency, Dead Letter

Description: Learn how to handle failures in Dapr-based batch processing using dead-letter topics, resiliency policies, and compensation workflows.

---

Batch processing jobs deal with large volumes of records, and some will inevitably fail due to invalid data, transient errors, or downstream outages. A robust failure handling strategy captures failed items, retries transient errors, and alerts on persistent failures without blocking the entire batch.

## Types of Batch Failures

1. **Transient failures**: Network timeouts, temporary service unavailability - should be retried.
2. **Data validation failures**: Invalid record format or business rule violations - should be quarantined.
3. **Poison messages**: Malformed messages that always fail - must be moved to dead-letter storage.
4. **Downstream failures**: Target system unavailable - should pause and retry later.

## Configure Resiliency for Retries

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: batch-resiliency
spec:
  policies:
    retries:
      batchRetry:
        policy: exponential
        maxRetries: 5
        initialInterval: 1s
        maxInterval: 30s
        multiplier: 2
    circuitBreakers:
      batchCircuitBreaker:
        maxRequests: 1
        interval: 30s
        timeout: 60s
        trip: consecutiveFailures >= 10
  targets:
    components:
      pubsub:
        outbound:
          retry: batchRetry
          circuitBreaker: batchCircuitBreaker
```

## Configure Dead-Letter Topics

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: batch-records-sub
spec:
  pubsubname: pubsub
  topic: batch-records
  route: /process-record
  deadLetterTopic: failed-batch-records
```

## Differentiate Error Types in Handlers

```python
from flask import Flask, request
import json

app = Flask(__name__)

@app.route('/process-record', methods=['POST'])
def process_record():
    event = request.json
    record = event['data']

    try:
        validate_record(record)
    except ValidationError as e:
        # Data error - do not retry, send to quarantine
        quarantine_record(record, str(e))
        return '', 200  # Acknowledge to prevent retry

    try:
        result = transform_and_store(record)
        return '', 200
    except TransientError as e:
        # Transient - return 500 to trigger Dapr retry
        print(f"Transient error for record {record['id']}: {e}")
        return '', 500
    except Exception as e:
        # Unknown error - log and quarantine
        quarantine_record(record, f"Unknown error: {e}")
        return '', 200
```

## Handle Dead-Letter Messages

Subscribe to the dead-letter topic for analysis and remediation:

```python
@app.route('/failed-batch-records', methods=['POST'])
def handle_dead_letter():
    event = request.json
    failed_record = event['data']

    # Store in quarantine with error context
    with DaprClient() as client:
        client.save_state(
            'statestore',
            f"quarantine:{failed_record.get('id', 'unknown')}:{int(time.time())}",
            json.dumps({
                "record": failed_record,
                "failedAt": datetime.utcnow().isoformat(),
                "deliveryAttempts": event.get('metadata', {}).get('deliveryCount', 'unknown')
            }),
            state_metadata={"ttlInSeconds": "2592000"}  # 30 days
        )

        # Alert operations team
        client.publish_event('pubsub', 'batch-alerts', {
            "type": "dead-letter",
            "recordId": failed_record.get('id'),
            "topic": "batch-records"
        })

    return '', 200
```

## Compensation Workflow for Partial Failures

When a batch partially succeeds, compensate for the failed portion:

```python
def batch_with_compensation_workflow(ctx: DaprWorkflowContext, batch: dict):
    processed = []
    failed = []

    for record in batch['records']:
        try:
            result = yield ctx.call_activity(process_single_record, input=record)
            processed.append(result)
        except Exception as e:
            failed.append({"record": record, "error": str(e)})

    if failed:
        # Compensate: roll back successfully processed records
        if batch.get('transactional'):
            for item in processed:
                yield ctx.call_activity(compensate_record, input=item)

        # Or: re-queue failed records for retry
        else:
            yield ctx.call_activity(requeue_failed_records, input={"failed": failed})

    return {"processed": len(processed), "failed": len(failed)}
```

## Alert on High Failure Rates

```python
def check_failure_rate(batch_id: str, total: int, failed: int):
    failure_rate = failed / total if total > 0 else 0

    if failure_rate > 0.1:  # More than 10% failure rate
        with DaprClient() as client:
            client.publish_event('pubsub', 'operations-alerts', {
                "severity": "warning",
                "batchId": batch_id,
                "message": f"High failure rate: {failure_rate:.1%}",
                "failed": failed,
                "total": total
            })
```

## Summary

Handling batch processing failures with Dapr requires distinguishing transient errors (return 500 for retry) from data errors (return 200 to prevent retry and quarantine). Configure dead-letter topics to capture poison messages that exhaust retries, and subscribe to the dead-letter topic for analysis and alerting. Use Dapr Workflow's compensation pattern to roll back or re-queue failed records when partial batch failure occurs.
