# How to Use ClickHouse with SQLAlchemy in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQLAlchemy, Python, Database, ORM

Description: Learn how to connect ClickHouse to SQLAlchemy in Python using clickhouse-sqlalchemy for querying, inserting, and modeling analytics data.

---

## Why SQLAlchemy with ClickHouse

SQLAlchemy is Python's most popular database toolkit. By integrating ClickHouse with SQLAlchemy, you can use familiar ORM patterns and raw SQL execution while leveraging ClickHouse's columnar speed for analytics workloads.

## Install Dependencies

```bash
pip install clickhouse-sqlalchemy sqlalchemy
```

## Create a Connection

```python
from sqlalchemy import create_engine

engine = create_engine(
    "clickhouse+http://default:@localhost:8123/default"
)
```

For native protocol:

```python
engine = create_engine(
    "clickhouse+native://default:@localhost:9000/default"
)
```

## Define a Table with ORM

```python
from sqlalchemy import Column, String, DateTime, Float
from sqlalchemy.orm import declarative_base
from clickhouse_sqlalchemy import engines

Base = declarative_base()

class Event(Base):
    __tablename__ = "events"
    event_id = Column(String, primary_key=True)
    user_id = Column(String)
    event_type = Column(String)
    ts = Column(DateTime)
    value = Column(Float)

    __table_args__ = (
        engines.MergeTree(order_by=["ts", "event_id"]),
    )
```

## Create Tables

```python
Base.metadata.create_all(engine)
```

## Insert Rows

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    session.add(Event(
        event_id="evt-001",
        user_id="user-42",
        event_type="click",
        ts="2026-01-01 10:00:00",
        value=1.0
    ))
    session.commit()
```

## Query with ORM

```python
from sqlalchemy.orm import Session

with Session(engine) as session:
    results = session.query(Event).filter(
        Event.event_type == "click"
    ).limit(10).all()
    for row in results:
        print(row.event_id, row.ts)
```

## Execute Raw SQL

```python
from sqlalchemy import text

with engine.connect() as conn:
    result = conn.execute(text(
        "SELECT event_type, count() AS cnt FROM events GROUP BY event_type"
    ))
    for row in result:
        print(row.event_type, row.cnt)
```

## Bulk Insert with Core API

```python
from sqlalchemy import insert

rows = [
    {"event_id": f"evt-{i}", "user_id": "u1", "event_type": "view", "ts": "2026-01-02 12:00:00", "value": float(i)}
    for i in range(1000)
]

with engine.connect() as conn:
    conn.execute(insert(Event), rows)
```

## Connection Pooling

```python
engine = create_engine(
    "clickhouse+http://default:@localhost:8123/default",
    pool_size=10,
    max_overflow=5,
    pool_timeout=30
)
```

## Summary

Using SQLAlchemy with ClickHouse lets you apply familiar Python patterns to analytical workloads. The `clickhouse-sqlalchemy` driver supports both HTTP and native protocols, ORM models with MergeTree engines, and raw SQL execution - making it easy to integrate ClickHouse into existing Python stacks.
