# How to Use ClickHouse with Python Type Hints

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Python, Type Hint, Type Safety, dataclass

Description: Use Python type hints, dataclasses, and typed query results to write safer ClickHouse code with better IDE support and runtime validation.

---

## Why Type Hints Matter with ClickHouse

ClickHouse query results are typically untyped tuples. Without type hints, you pass raw values around and catch type errors only at runtime. Combining type hints with dataclasses and typed result parsing makes ClickHouse code more maintainable and refactor-friendly.

## Typed Query Results with dataclasses

Define a dataclass that mirrors a ClickHouse row:

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class EventRow:
    user_id: int
    event_time: datetime
    event_type: str
    value: float
    session_id: Optional[str] = None
```

Parse query results into typed objects:

```python
import clickhouse_connect
from typing import List

def get_events(client, date: str) -> List[EventRow]:
    result = client.query(
        f"SELECT user_id, event_time, event_type, value FROM events "
        f"WHERE toDate(event_time) = '{date}'"
    )
    return [
        EventRow(
            user_id=row[0],
            event_time=row[1],
            event_type=row[2],
            value=row[3]
        )
        for row in result.result_rows
    ]
```

## Typed Insert Functions

```python
def insert_events(client, events: List[EventRow]) -> None:
    rows = [
        [e.user_id, e.event_time, e.event_type, e.value]
        for e in events
    ]
    client.insert(
        "events",
        rows,
        column_names=["user_id", "event_time", "event_type", "value"]
    )
```

## Using TypedDict for Query Parameters

```python
from typing import TypedDict

class EventFilter(TypedDict):
    start_date: str
    end_date: str
    event_types: List[str]

def query_events(client, filters: EventFilter) -> List[EventRow]:
    types_list = ", ".join(f"'{t}'" for t in filters["event_types"])
    result = client.query(f"""
        SELECT user_id, event_time, event_type, value FROM events
        WHERE event_time BETWEEN '{filters["start_date"]}' AND '{filters["end_date"]}'
          AND event_type IN ({types_list})
    """)
    return [EventRow(*row) for row in result.result_rows]
```

## Protocol-Based Client Abstraction

Define a Protocol to abstract the ClickHouse client for easier testing:

```python
from typing import Protocol, Any

class ClickHouseClientProtocol(Protocol):
    def query(self, sql: str) -> Any: ...
    def command(self, sql: str) -> None: ...
    def insert(self, table: str, data: list, column_names: list) -> None: ...

def get_user_count(client: ClickHouseClientProtocol, date: str) -> int:
    result = client.query(
        f"SELECT count(DISTINCT user_id) FROM events WHERE toDate(event_time) = '{date}'"
    )
    return int(result.first_row[0])
```

## Running mypy

```bash
pip install mypy
mypy clickhouse_service.py --strict
```

mypy will catch type mismatches, missing Optional handling, and incorrect return types at development time rather than in production.

## Summary

Using Python type hints with ClickHouse means defining dataclasses for row types, TypedDicts for filter parameters, and Protocol classes for client abstraction. This enables IDE autocompletion, mypy static analysis, and makes refactoring ClickHouse query code much safer across large codebases.
