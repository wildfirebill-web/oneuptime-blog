# How to Build Async Query Endpoints for ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Async Query, Job Queue, API Design, Long-Running Query

Description: Build async query endpoints for ClickHouse that submit long-running queries as background jobs and return results via polling or webhooks.

---

## When to Use Async Queries

Synchronous API endpoints with 30-second timeouts cannot handle ClickHouse queries that run for minutes (large exports, complex cross-shard aggregations, backfill reports). Async query APIs submit the job immediately, return a job ID, and let clients poll for completion.

## Architecture

```text
Client -> POST /queries -> job submitted, returns job_id
Client -> GET  /queries/{job_id} -> returns status (pending/running/done/failed)
Client -> GET  /queries/{job_id}/results -> returns data when done
```

## Job Storage Table

```sql
-- ClickHouse query_jobs table (in PostgreSQL or Redis in production)
CREATE TABLE query_jobs
(
    job_id UUID DEFAULT generateUUIDv4(),
    submitted_at DateTime DEFAULT now(),
    completed_at DateTime,
    status LowCardinality(String) DEFAULT 'pending',
    query_text String,
    result_rows UInt64,
    error_message String
)
ENGINE = MergeTree
ORDER BY (submitted_at, job_id);
```

## Express Async Query API

```javascript
// app.js
const express = require('express');
const { v4: uuidv4 } = require('uuid');
const client = require('./db');

const app = express();
app.use(express.json());

// In-memory job store (use Redis or PostgreSQL in production)
const jobs = new Map();

// Submit a query job
app.post('/api/queries', async (req, res) => {
  const { query, params = {} } = req.body;

  if (!query) {
    return res.status(400).json({ error: 'query is required' });
  }

  const jobId = uuidv4();
  jobs.set(jobId, {
    id: jobId,
    status: 'pending',
    submitted_at: new Date().toISOString(),
    query,
    params,
  });

  // Run query in background
  runQueryAsync(jobId, query, params);

  res.status(202).json({
    job_id: jobId,
    status: 'pending',
    poll_url: `/api/queries/${jobId}`,
  });
});

async function runQueryAsync(jobId, query, params) {
  const job = jobs.get(jobId);
  job.status = 'running';
  job.started_at = new Date().toISOString();

  try {
    const result = await client.query({
      query,
      query_params: params,
      format: 'JSONEachRow',
    });
    const data = await result.json();

    job.status = 'done';
    job.completed_at = new Date().toISOString();
    job.result = data;
    job.row_count = data.length;
  } catch (err) {
    job.status = 'failed';
    job.completed_at = new Date().toISOString();
    job.error = err.message;
  }
}

// Poll for job status
app.get('/api/queries/:jobId', (req, res) => {
  const job = jobs.get(req.params.jobId);
  if (!job) return res.status(404).json({ error: 'Job not found' });

  const { result, ...jobMeta } = job;  // exclude result from status endpoint
  res.json(jobMeta);
});

// Fetch results when complete
app.get('/api/queries/:jobId/results', (req, res) => {
  const job = jobs.get(req.params.jobId);
  if (!job) return res.status(404).json({ error: 'Job not found' });
  if (job.status !== 'done') {
    return res.status(409).json({ error: 'Query not yet complete', status: job.status });
  }

  res.json({
    job_id: job.id,
    row_count: job.row_count,
    data: job.result,
  });
});
```

## Client Usage

```bash
# Submit query
JOB=$(curl -s -X POST http://localhost:3000/api/queries \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT count() FROM events WHERE event_time >= today() - 365"}')
echo $JOB

# Poll until complete
JOB_ID=$(echo $JOB | jq -r '.job_id')
while true; do
  STATUS=$(curl -s "http://localhost:3000/api/queries/$JOB_ID" | jq -r '.status')
  echo "Status: $STATUS"
  if [ "$STATUS" = "done" ] || [ "$STATUS" = "failed" ]; then break; fi
  sleep 2
done

# Fetch results
curl "http://localhost:3000/api/queries/$JOB_ID/results"
```

## Summary

Async query endpoints for ClickHouse decouple query submission from result retrieval. Submit jobs to a background worker, store job state with a unique ID, and let clients poll for completion. For production use, replace the in-memory job store with Redis or PostgreSQL and use a proper job queue like Bull or BullMQ to manage worker concurrency and retries.
