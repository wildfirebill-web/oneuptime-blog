# How to Build a Python ETL Pipeline for ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ETL, Python, Data Pipeline, Data Engineering

Description: Build a production-grade Python ETL pipeline that extracts data from a source, transforms it, and loads it into ClickHouse with error handling and idempotency.

---

## ETL Pipeline Architecture

A typical ClickHouse ETL pipeline has three stages:

1. Extract - pull data from source (API, database, files)
2. Transform - clean, normalize, enrich
3. Load - bulk insert into ClickHouse

The key non-functional requirements are idempotency (re-running doesn't duplicate data), error handling (partial failures don't corrupt data), and observability (you know what ran and when).

## Project Structure

```text
etl/
  __init__.py
  extract.py
  transform.py
  load.py
  pipeline.py
  config.py
```

## Extract Stage

```python
# extract.py
import requests
from datetime import date

def extract_events(api_url: str, for_date: date) -> list[dict]:
    response = requests.get(
        f"{api_url}/events",
        params={"date": for_date.isoformat()},
        timeout=30
    )
    response.raise_for_status()
    return response.json()["events"]
```

## Transform Stage

```python
# transform.py
from datetime import datetime
from typing import Any

def transform_events(raw_events: list[dict]) -> list[dict]:
    cleaned = []
    for event in raw_events:
        try:
            cleaned.append({
                "user_id": int(event["userId"]),
                "event_time": datetime.fromisoformat(event["timestamp"]),
                "event_type": event.get("type", "unknown").lower().strip(),
                "value": float(event.get("value", 0.0)),
            })
        except (KeyError, ValueError, TypeError):
            # Log and skip malformed records
            continue
    return cleaned
```

## Load Stage

```python
# load.py
import clickhouse_connect
from typing import Any

def load_events(client, events: list[dict], table: str = "events") -> int:
    if not events:
        return 0

    client.insert(
        table,
        [[e["user_id"], e["event_time"], e["event_type"], e["value"]] for e in events],
        column_names=["user_id", "event_time", "event_type", "value"]
    )
    return len(events)
```

## Pipeline Orchestration with Idempotency

```python
# pipeline.py
import clickhouse_connect
from datetime import date, timedelta
from extract import extract_events
from transform import transform_events
from load import load_events

def run_pipeline(for_date: date, api_url: str, ch_host: str):
    client = clickhouse_connect.get_client(host=ch_host)

    # Check if we already processed this date (idempotency check)
    result = client.query(
        "SELECT count() FROM etl_runs WHERE run_date = %(date)s AND status = 'success'",
        parameters={"date": for_date}
    )
    if result.first_row[0] > 0:
        print(f"Already processed {for_date}, skipping")
        return

    raw = extract_events(api_url, for_date)
    transformed = transform_events(raw)
    loaded = load_events(client, transformed)

    # Record completion
    client.insert("etl_runs", [[for_date, "success", loaded]], column_names=["run_date", "status", "rows"])
    print(f"Pipeline complete for {for_date}: {loaded} rows loaded")

if __name__ == "__main__":
    run_pipeline(date.today() - timedelta(days=1), "https://api.example.com", "localhost")
```

## Summary

A ClickHouse ETL pipeline separates extraction, transformation, and loading into distinct stages. Idempotency checks via a run log table prevent duplicate inserts on retries. Transform errors are logged and skipped rather than failing the entire batch. Bulk inserts via `clickhouse-connect` ensure efficient loading.
