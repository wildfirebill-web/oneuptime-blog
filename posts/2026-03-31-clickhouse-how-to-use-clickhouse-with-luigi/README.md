# How to Use ClickHouse with Luigi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Luigi, Pipeline, ETL, Task, Dependency

Description: Use Luigi to build dependency-aware ETL pipelines that load and transform data in ClickHouse, with automatic task scheduling and failure handling.

---

Luigi is a Python library from Spotify for building data pipelines with dependency resolution. Each step is a Task with defined inputs and outputs. When connected to ClickHouse, Luigi orchestrates data flows into your analytical database.

## Installing Dependencies

```bash
pip install luigi clickhouse-connect pandas
```

## Defining a Luigi Task that Writes to ClickHouse

Create `pipeline.py`:

```python
import luigi
import clickhouse_connect
import pandas as pd
from datetime import date

class ExtractEventsTask(luigi.Task):
    date = luigi.DateParameter(default=date.today())

    def output(self):
        return luigi.LocalTarget(f"/tmp/events_{self.date}.parquet")

    def run(self):
        # Simulate extraction from external source
        df = pd.DataFrame({
            'user_id': range(100),
            'event_type': ['click'] * 50 + ['view'] * 50,
            'event_date': [str(self.date)] * 100
        })
        df.to_parquet(self.output().path, index=False)


class LoadToClickHouseTask(luigi.Task):
    date = luigi.DateParameter(default=date.today())

    def requires(self):
        return ExtractEventsTask(date=self.date)

    def output(self):
        return luigi.LocalTarget(f"/tmp/clickhouse_loaded_{self.date}.marker")

    def run(self):
        df = pd.read_parquet(self.input().path)
        client = clickhouse_connect.get_client(
            host='localhost',
            port=8123,
            username='default',
            password=''
        )
        client.insert_df('default.events_staging', df)
        with self.output().open('w') as f:
            f.write(f"Loaded {len(df)} rows")
```

## Running the Pipeline

```bash
python -m luigi --module pipeline LoadToClickHouseTask --date 2024-01-15 --local-scheduler
```

Or use the Luigi central scheduler for production:

```bash
luigid &
python -m luigi --module pipeline LoadToClickHouseTask --date 2024-01-15
```

## Adding a Transformation Task in ClickHouse SQL

```python
class TransformEventsTask(luigi.Task):
    date = luigi.DateParameter(default=date.today())

    def requires(self):
        return LoadToClickHouseTask(date=self.date)

    def output(self):
        return luigi.LocalTarget(f"/tmp/transformed_{self.date}.marker")

    def run(self):
        client = clickhouse_connect.get_client(
            host='localhost', port=8123, username='default', password=''
        )
        client.command(f"""
            INSERT INTO analytics.daily_summary
            SELECT
                event_date,
                event_type,
                count() AS events,
                uniq(user_id) AS unique_users
            FROM default.events_staging
            WHERE event_date = '{self.date}'
            GROUP BY event_date, event_type
        """)
        with self.output().open('w') as f:
            f.write("done")
```

## Summary

Luigi's task-dependency model pairs well with ClickHouse ETL workflows. Each task is a discrete, retriable unit of work. Dependencies ensure that data is loaded before transformations run. ClickHouse handles the heavy lifting of aggregation and storage, while Luigi manages the pipeline orchestration.
