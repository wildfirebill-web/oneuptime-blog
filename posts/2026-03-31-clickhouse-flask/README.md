# How to Use ClickHouse with Flask

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Flask, Python, Database, Analytics, Api

Description: Build a Flask analytics API backed by ClickHouse using clickhouse-connect, application factory pattern, blueprints, and parameterized queries for safe data access.

---

Flask's minimal design makes it easy to add ClickHouse as a data source without fighting a framework's assumptions. This guide covers the full integration: connection management with the application factory pattern, blueprints for route organization, and production-ready query patterns.

## Installation

```bash
pip install flask clickhouse-connect python-dotenv
```

## Project Structure

```text
app/
  __init__.py
  clickhouse.py
  blueprints/
    events.py
    analytics.py
config.py
run.py
.env
requirements.txt
```

## Configuration

```python
# config.py
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    CLICKHOUSE_HOST     = os.getenv("CLICKHOUSE_HOST", "localhost")
    CLICKHOUSE_PORT     = int(os.getenv("CLICKHOUSE_PORT", 8123))
    CLICKHOUSE_USER     = os.getenv("CLICKHOUSE_USER", "default")
    CLICKHOUSE_PASSWORD = os.getenv("CLICKHOUSE_PASSWORD", "")
    CLICKHOUSE_DATABASE = os.getenv("CLICKHOUSE_DATABASE", "analytics")
    CLICKHOUSE_SECURE   = os.getenv("CLICKHOUSE_SECURE", "false").lower() == "true"

class ProductionConfig(Config):
    CLICKHOUSE_SECURE = True
```

Environment file:

```text
CLICKHOUSE_HOST=localhost
CLICKHOUSE_PORT=8123
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=
CLICKHOUSE_DATABASE=analytics
```

## ClickHouse Extension

Flask uses extensions to share resources across blueprints. Create a ClickHouse extension:

```python
# app/clickhouse.py
import clickhouse_connect
from clickhouse_connect.driver.client import Client

class ClickHouse:
    def __init__(self, app=None):
        self.app = app
        self._client: Client | None = None
        if app is not None:
            self.init_app(app)

    def init_app(self, app):
        app.config.setdefault("CLICKHOUSE_HOST", "localhost")
        app.config.setdefault("CLICKHOUSE_PORT", 8123)
        app.config.setdefault("CLICKHOUSE_USER", "default")
        app.config.setdefault("CLICKHOUSE_PASSWORD", "")
        app.config.setdefault("CLICKHOUSE_DATABASE", "default")
        app.config.setdefault("CLICKHOUSE_SECURE", False)
        app.extensions["clickhouse"] = self

    @property
    def client(self) -> Client:
        if self._client is None:
            from flask import current_app
            cfg = current_app.config
            self._client = clickhouse_connect.get_client(
                host=cfg["CLICKHOUSE_HOST"],
                port=cfg["CLICKHOUSE_PORT"],
                username=cfg["CLICKHOUSE_USER"],
                password=cfg["CLICKHOUSE_PASSWORD"],
                database=cfg["CLICKHOUSE_DATABASE"],
                secure=cfg["CLICKHOUSE_SECURE"],
                compress=True,
            )
        return self._client

    def query(self, sql: str, parameters: dict = None):
        return self.client.query(sql, parameters=parameters)

    def insert(self, table: str, data: list, column_names: list):
        return self.client.insert(table, data, column_names=column_names)


ch = ClickHouse()
```

## Application Factory

```python
# app/__init__.py
from flask import Flask
from config import Config
from app.clickhouse import ch

def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)

    ch.init_app(app)

    from app.blueprints.events import events_bp
    from app.blueprints.analytics import analytics_bp

    app.register_blueprint(events_bp)
    app.register_blueprint(analytics_bp)

    @app.route("/health")
    def health():
        result = ch.query("SELECT 1 AS ok")
        return {"status": "ok" if result.first_row[0] == 1 else "error"}

    return app
```

## Events Blueprint

```python
# app/blueprints/events.py
from flask import Blueprint, request, jsonify, abort
from datetime import datetime
from app.clickhouse import ch
import json

events_bp = Blueprint("events", __name__, url_prefix="/events")

@events_bp.post("/")
def ingest():
    body = request.get_json(force=True)
    required = ("user_id", "event_type", "page")
    if not all(k in body for k in required):
        abort(400, description=f"Missing required fields: {required}")

    ch.insert(
        "analytics.events",
        [[
            int(body["user_id"]),
            body.get("session_id", ""),
            body["event_type"],
            body["page"],
            json.dumps(body.get("properties", {})),
            datetime.utcnow(),
        ]],
        column_names=["user_id", "session_id", "event_type", "page", "properties", "ts"],
    )
    return jsonify({"status": "accepted"}), 201


@events_bp.post("/batch")
def ingest_batch():
    body = request.get_json(force=True)
    if not isinstance(body, list):
        abort(400, description="Expected a JSON array")

    rows = []
    for item in body:
        rows.append([
            int(item["user_id"]),
            item.get("session_id", ""),
            item["event_type"],
            item["page"],
            json.dumps(item.get("properties", {})),
            datetime.utcnow(),
        ])

    ch.insert(
        "analytics.events",
        rows,
        column_names=["user_id", "session_id", "event_type", "page", "properties", "ts"],
    )
    return jsonify({"status": "accepted", "count": len(rows)}), 201


@events_bp.get("/")
def list_events():
    event_type = request.args.get("event_type")
    user_id    = request.args.get("user_id", type=int)
    limit      = min(request.args.get("limit", 100, type=int), 1000)

    conditions = ["1=1"]
    params: dict = {"limit": limit}

    if event_type:
        conditions.append("event_type = {event_type:String}")
        params["event_type"] = event_type
    if user_id:
        conditions.append("user_id = {user_id:UInt64}")
        params["user_id"] = user_id

    where = " AND ".join(conditions)
    result = ch.query(
        f"""
        SELECT event_id, user_id, event_type, page, ts
        FROM analytics.events
        WHERE {where}
        ORDER BY ts DESC
        LIMIT {{limit:UInt32}}
        """,
        parameters=params,
    )

    events = [
        {"event_id": str(r[0]), "user_id": r[1], "event_type": r[2], "page": r[3], "ts": str(r[4])}
        for r in result.result_rows
    ]
    return jsonify(events)
```

## Analytics Blueprint

```python
# app/blueprints/analytics.py
from flask import Blueprint, request, jsonify
from datetime import datetime, timedelta
from app.clickhouse import ch

analytics_bp = Blueprint("analytics", __name__, url_prefix="/analytics")


@analytics_bp.get("/summary")
def summary():
    days  = request.args.get("days", 7, type=int)
    since = datetime.utcnow() - timedelta(days=days)

    result = ch.query(
        """
        SELECT
            event_type,
            count()       AS total,
            uniq(user_id) AS users,
            min(ts)       AS first_seen,
            max(ts)       AS last_seen
        FROM analytics.events
        WHERE ts >= {since:DateTime}
        GROUP BY event_type
        ORDER BY total DESC
        """,
        parameters={"since": since},
    )

    return jsonify([
        {
            "event_type": r[0],
            "total":      r[1],
            "users":      r[2],
            "first_seen": str(r[3]),
            "last_seen":  str(r[4]),
        }
        for r in result.result_rows
    ])


@analytics_bp.get("/top-pages")
def top_pages():
    days  = request.args.get("days", 7, type=int)
    limit = min(request.args.get("limit", 10, type=int), 100)
    since = datetime.utcnow() - timedelta(days=days)

    result = ch.query(
        """
        SELECT
            page,
            count()       AS views,
            uniq(user_id) AS unique_users
        FROM analytics.events
        WHERE event_type = 'page_view'
          AND ts >= {since:DateTime}
        GROUP BY page
        ORDER BY views DESC
        LIMIT {limit:UInt32}
        """,
        parameters={"since": since, "limit": limit},
    )

    return jsonify([
        {"page": r[0], "views": r[1], "unique_users": r[2]}
        for r in result.result_rows
    ])


@analytics_bp.get("/active-users")
def active_users():
    result = ch.query(
        """
        SELECT
            toDate(ts)    AS day,
            uniq(user_id) AS dau
        FROM analytics.events
        WHERE ts >= today() - 30
        GROUP BY day
        ORDER BY day
        """
    )

    return jsonify([
        {"day": str(r[0]), "dau": r[1]}
        for r in result.result_rows
    ])


@analytics_bp.get("/session-duration")
def session_duration():
    since = datetime.utcnow() - timedelta(days=7)

    result = ch.query(
        """
        SELECT
            session_id,
            dateDiff('second', min(ts), max(ts)) AS duration_seconds,
            count() AS event_count
        FROM analytics.events
        WHERE ts >= {since:DateTime}
          AND session_id != ''
        GROUP BY session_id
        HAVING duration_seconds > 0
        ORDER BY duration_seconds DESC
        LIMIT 100
        """,
        parameters={"since": since},
    )

    durations = [r[1] for r in result.result_rows]
    if not durations:
        return jsonify({"p50": 0, "p90": 0, "p99": 0})

    durations.sort()
    n = len(durations)
    return jsonify({
        "p50": durations[int(n * 0.50)],
        "p90": durations[int(n * 0.90)],
        "p99": durations[int(n * 0.99)],
        "sample_count": n,
    })
```

## Entry Point

```python
# run.py
from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=5000)
```

## Running the Application

```bash
flask run --host=0.0.0.0 --port=5000
```

## Testing with pytest

```python
# tests/test_analytics.py
import pytest
from app import create_app

@pytest.fixture
def client():
    app = create_app()
    app.config["TESTING"] = True
    with app.test_client() as c:
        yield c

def test_health(client):
    resp = client.get("/health")
    assert resp.status_code == 200
    assert resp.json["status"] == "ok"

def test_summary(client):
    resp = client.get("/analytics/summary?days=1")
    assert resp.status_code == 200
    assert isinstance(resp.json, list)
```

## Error Handling

Register a global error handler to return consistent JSON errors:

```python
# app/__init__.py (add inside create_app)
from flask import jsonify

@app.errorhandler(400)
def bad_request(e):
    return jsonify({"error": str(e.description)}), 400

@app.errorhandler(500)
def server_error(e):
    return jsonify({"error": "Internal server error"}), 500
```

## Summary

Flask integrates with ClickHouse through a lightweight extension that wraps `clickhouse-connect`. The application factory pattern initializes the connection once, blueprints keep routes organized, and parameterized queries prevent SQL injection. For production, add connection retry logic, set `CLICKHOUSE_SECURE=true`, and configure `max_execution_time` to prevent runaway queries from blocking the Flask worker processes.
