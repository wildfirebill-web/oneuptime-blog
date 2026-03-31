# How to Use the Dapr Jobs API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Jobs API, Scheduling, Microservice, Distributed System

Description: Learn how to use the Dapr Jobs API to schedule and manage recurring jobs in your microservices with a persistent, distributed job scheduler.

---

## Overview

The Dapr Jobs API provides a built-in job scheduling system that lets your microservices schedule one-time or recurring tasks without depending on external schedulers. Jobs are stored durably in Dapr's scheduler service and survive service restarts. When a job fires, Dapr invokes a callback endpoint in your application.

## Prerequisites

- Dapr 1.13 or later installed
- A running application with a Dapr sidecar
- Dapr Scheduler service running (included in standard Dapr installation)

## Scheduling a Job via HTTP

Create a one-time job that fires after a delay:

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/jobs/send-reminder \
  -H "Content-Type: application/json" \
  -d '{
    "job": {
      "schedule": "@every 30s",
      "data": {
        "@type": "type.googleapis.com/google.protobuf.StringValue",
        "value": "{\"userId\": \"u123\", \"message\": \"Your trial expires soon\"}"
      },
      "ttl": "5m"
    }
  }'
```

Create a recurring cron-based job:

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/jobs/daily-report \
  -H "Content-Type: application/json" \
  -d '{
    "job": {
      "schedule": "0 8 * * *",
      "repeats": 0,
      "data": {
        "@type": "type.googleapis.com/google.protobuf.StringValue",
        "value": "{\"reportType\": \"daily-summary\"}"
      }
    }
  }'
```

## Handling Job Callbacks in Your Application

Dapr calls a specific endpoint on your application when a job fires. Register a handler at `/job/{jobName}`:

### Node.js with Express

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// Handler for the 'daily-report' job
app.post('/job/daily-report', async (req, res) => {
    console.log('Daily report job triggered:', req.body);
    const data = JSON.parse(req.body.value);

    try {
        await generateDailyReport(data.reportType);
        console.log('Report generated successfully');
        res.sendStatus(200);
    } catch (err) {
        console.error('Failed to generate report:', err);
        res.sendStatus(500);
    }
});

app.listen(3000, () => console.log('App listening on port 3000'));
```

### Python with Flask

```python
from flask import Flask, request, jsonify
import json

app = Flask(__name__)

@app.route('/job/daily-report', methods=['POST'])
def handle_daily_report():
    body = request.get_json()
    data = json.loads(body.get('value', '{}'))
    report_type = data.get('reportType')

    print(f'Job triggered: daily-report for type={report_type}')

    try:
        generate_report(report_type)
        return '', 200
    except Exception as e:
        print(f'Error: {e}')
        return str(e), 500

if __name__ == '__main__':
    app.run(port=3000)
```

## Managing Jobs with the .NET SDK

```csharp
using Dapr.Client;
using Google.Protobuf.WellKnownTypes;

var client = new DaprClientBuilder().Build();

// Schedule a job
var job = new DaprJobSchedule("0 9 * * MON-FRI");  // Weekdays at 9am
await client.ScheduleJobAsync(
    jobName: "weekly-invoice",
    schedule: job,
    payload: new StringValue { Value = "{\"customerId\": \"c456\"}" }
);

Console.WriteLine("Job scheduled");

// Get job details
var jobDetails = await client.GetJobAsync("weekly-invoice");
Console.WriteLine($"Next run: {jobDetails.Schedule}");

// Delete a job
await client.DeleteJobAsync("weekly-invoice");
Console.WriteLine("Job deleted");
```

## Managing Jobs with the Python SDK

```python
from dapr.clients import DaprClient
from dapr.clients.grpc._jobs import Job

with DaprClient() as client:
    # Schedule a job
    job = Job(
        name="send-reminder",
        schedule="@every 1h",
        data=b'{"userId": "u789", "message": "Reminder"}',
        repeats=24  # Run 24 times then stop
    )
    client.schedule_job(job)
    print("Job scheduled")

    # Get job status
    details = client.get_job("send-reminder")
    print(f"Job schedule: {details.schedule}")

    # Delete job
    client.delete_job("send-reminder")
    print("Job deleted")
```

## Supported Schedule Formats

| Format | Example | Meaning |
|--------|---------|---------|
| Cron | `0 8 * * *` | Every day at 8am |
| Period | `@every 30m` | Every 30 minutes |
| Once | `@once` | Run once immediately |
| Named | `@daily` | Every day at midnight |
| Named | `@weekly` | Every Sunday at midnight |

## Running with the Dapr CLI

Start your app with the Dapr sidecar:

```bash
dapr run \
  --app-id jobservice \
  --app-port 3000 \
  --dapr-http-port 3500 \
  -- node app.js
```

Test by manually triggering the job handler:

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/jobs/daily-report
```

## Listing and Querying Jobs

Get all jobs for the current application:

```bash
curl http://localhost:3500/v1.0-alpha1/jobs/daily-report
```

Response:

```json
{
  "job": {
    "name": "daily-report",
    "schedule": "0 8 * * *",
    "repeats": 0,
    "dueTime": "2026-04-01T08:00:00Z"
  }
}
```

## Summary

The Dapr Jobs API provides durable job scheduling without requiring an external cron system or queue. You schedule jobs via the Dapr sidecar HTTP API or SDK, define a callback route in your application at `/job/{jobName}`, and Dapr invokes it when the job fires. Jobs support cron expressions, period intervals, and repeat limits, and their state survives service restarts because Dapr persists them in the scheduler service.
