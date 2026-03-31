# How to Use ClickHouse with Mage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Mage, ETL, Data Pipeline, Orchestration, Python

Description: Build interactive ETL pipelines with Mage that read from sources, transform data, and load results into ClickHouse for fast analytical queries.

---

Mage is a modern data pipeline tool with an interactive notebook-like interface. It lets you build, test, and run ETL pipelines visually while keeping logic in Python or SQL. ClickHouse serves as a high-performance destination for analytical data.

## Starting Mage

```bash
pip install mage-ai
mage start my_pipeline_project
```

Open `http://localhost:6789` in your browser.

## Adding a ClickHouse Connection

In Mage's IO config file (`io_config.yaml`), add your ClickHouse credentials:

```text
version: "1.0"

default:
  CLICKHOUSE_DATABASE: default
  CLICKHOUSE_HOST: localhost
  CLICKHOUSE_INTERFACE: http
  CLICKHOUSE_PASSWORD: ""
  CLICKHOUSE_PORT: 8123
  CLICKHOUSE_USERNAME: default
```

## Creating a Data Loader Block

Add a Python data loader block to extract data:

```python
import clickhouse_connect
import pandas as pd
from mage_ai.settings.repo import get_repo_path
from mage_ai.io.config import ConfigFileLoader
from os import path

if 'data_loader' not in globals():
    from mage_ai.data_preparation.decorators import data_loader

@data_loader
def load_data(*args, **kwargs):
    client = clickhouse_connect.get_client(
        host='localhost',
        port=8123,
        username='default',
        password=''
    )
    result = client.query("""
        SELECT user_id, event_type, created_at
        FROM raw.events
        WHERE toDate(created_at) = yesterday()
    """)
    return pd.DataFrame(result.result_rows, columns=result.column_names)
```

## Creating a Transformer Block

```python
import pandas as pd

if 'transformer' not in globals():
    from mage_ai.data_preparation.decorators import transformer

@transformer
def transform(df: pd.DataFrame, *args, **kwargs):
    summary = df.groupby('event_type').agg(
        count=('user_id', 'count'),
        unique_users=('user_id', 'nunique')
    ).reset_index()
    summary['date'] = pd.Timestamp.today().date()
    return summary
```

## Creating a Data Exporter Block

```python
import clickhouse_connect
import pandas as pd

if 'data_exporter' not in globals():
    from mage_ai.data_preparation.decorators import data_exporter

@data_exporter
def export_data(df: pd.DataFrame, *args, **kwargs):
    client = clickhouse_connect.get_client(
        host='localhost',
        port=8123,
        username='default',
        password=''
    )
    client.insert_df('analytics.event_summary', df)
    print(f"Exported {len(df)} rows to ClickHouse")
```

## Using the Built-in ClickHouse Connector

Mage also has a native ClickHouse integration:

```python
from mage_ai.io.clickhouse import ClickHouse

config_path = path.join(get_repo_path(), 'io_config.yaml')
loader = ConfigFileLoader(config_path, 'default')

with ClickHouse.with_config(loader) as ch:
    df = ch.load("SELECT * FROM analytics.event_summary LIMIT 100")
```

## Summary

Mage's visual pipeline builder combined with ClickHouse makes it easy to build ETL workflows with real-time feedback. You can test individual blocks interactively, chain them into pipelines, and schedule runs - all while ClickHouse handles fast data storage and transformation on the query side.
