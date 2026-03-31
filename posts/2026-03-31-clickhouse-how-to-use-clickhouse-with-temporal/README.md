# How to Use ClickHouse with Temporal

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Temporal, Workflow, ETL, Activity, Orchestration

Description: Use Temporal workflows and activities to build reliable, retryable data pipelines that load and query ClickHouse as part of long-running business processes.

---

Temporal is a workflow orchestration platform that handles retries, timeouts, and state for long-running processes. Using Temporal to manage ClickHouse data operations gives you durability and observability for complex ETL and analytical workflows.

## Installing Dependencies

```bash
pip install temporalio clickhouse-connect pandas
```

## Defining Activities

Activities in Temporal are the individual steps of a workflow. Create `activities.py`:

```python
import clickhouse_connect
import pandas as pd
from temporalio import activity

@activity.defn
async def load_events_to_clickhouse(date: str) -> int:
    client = clickhouse_connect.get_client(
        host='localhost',
        port=8123,
        username='default',
        password=''
    )
    # Simulate loading data
    df = pd.DataFrame({
        'event_date': [date] * 10,
        'event_type': ['click'] * 5 + ['view'] * 5,
        'user_id': range(10)
    })
    client.insert_df('default.events_staging', df)
    return len(df)

@activity.defn
async def aggregate_daily_events(date: str) -> None:
    client = clickhouse_connect.get_client(
        host='localhost',
        port=8123,
        username='default',
        password=''
    )
    client.command(f"""
        INSERT INTO analytics.daily_summary
        SELECT
            event_date,
            event_type,
            count() AS events,
            uniq(user_id) AS unique_users
        FROM default.events_staging
        WHERE event_date = '{date}'
        GROUP BY event_date, event_type
    """)
```

## Defining the Workflow

Create `workflows.py`:

```python
from datetime import timedelta
from temporalio import workflow
from temporalio.common import RetryPolicy
from activities import load_events_to_clickhouse, aggregate_daily_events

@workflow.defn
class ClickHouseETLWorkflow:
    @workflow.run
    async def run(self, date: str) -> str:
        retry_policy = RetryPolicy(
            maximum_attempts=3,
            initial_interval=timedelta(seconds=5)
        )

        row_count = await workflow.execute_activity(
            load_events_to_clickhouse,
            date,
            start_to_close_timeout=timedelta(minutes=10),
            retry_policy=retry_policy
        )

        await workflow.execute_activity(
            aggregate_daily_events,
            date,
            start_to_close_timeout=timedelta(minutes=5),
            retry_policy=retry_policy
        )

        return f"Processed {row_count} rows for {date}"
```

## Running a Worker

```python
import asyncio
from temporalio.client import Client
from temporalio.worker import Worker
from workflows import ClickHouseETLWorkflow
from activities import load_events_to_clickhouse, aggregate_daily_events

async def main():
    client = await Client.connect("localhost:7233")
    worker = Worker(
        client,
        task_queue="clickhouse-etl",
        workflows=[ClickHouseETLWorkflow],
        activities=[load_events_to_clickhouse, aggregate_daily_events]
    )
    await worker.run()

asyncio.run(main())
```

## Triggering the Workflow

```python
async def trigger():
    client = await Client.connect("localhost:7233")
    handle = await client.start_workflow(
        ClickHouseETLWorkflow.run,
        "2024-01-15",
        id="etl-2024-01-15",
        task_queue="clickhouse-etl"
    )
    result = await handle.result()
    print(result)
```

## Summary

Temporal brings durability and fault tolerance to ClickHouse data pipelines. Each activity can be retried independently on failure, workflows persist their state across worker restarts, and the Temporal UI provides full visibility into workflow history. This is especially useful for multi-step ETL processes where partial failures are common.
