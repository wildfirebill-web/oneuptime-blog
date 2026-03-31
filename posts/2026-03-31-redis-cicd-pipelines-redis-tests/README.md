# How to Set Up CI/CD Pipelines with Redis Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CI/CD, Testing, GitHub Action, Docker

Description: Configure CI/CD pipelines that spin up Redis as a service container for integration tests, covering GitHub Actions, GitLab CI, and environment variable setup.

---

Running Redis integration tests in CI requires a Redis instance that is available for the duration of the test job and torn down automatically after. Most CI platforms support service containers - a Docker-based Redis instance that runs alongside your test job.

## GitHub Actions

```yaml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        env:
          REDIS_HOST: localhost
          REDIS_PORT: 6379
        run: pytest tests/ -v
```

The `health-cmd` ensures GitHub Actions waits for Redis to be ready before running tests.

## GitLab CI

```yaml
test:
  image: python:3.12
  services:
    - name: redis:7
      alias: redis
  variables:
    REDIS_HOST: redis
    REDIS_PORT: "6379"
  script:
    - pip install -r requirements.txt
    - pytest tests/ -v
```

In GitLab CI, the service is reachable at the alias hostname (`redis`).

## CircleCI

```yaml
version: 2.1

jobs:
  test:
    docker:
      - image: cimg/python:3.12
      - image: redis:7
        command: redis-server --save "" --appendonly no

    environment:
      REDIS_HOST: localhost
      REDIS_PORT: "6379"

    steps:
      - checkout
      - run: pip install -r requirements.txt
      - run: pytest tests/ -v
```

## Application Test Configuration

Read Redis connection details from environment variables in your tests:

```python
import os
import redis
import pytest

@pytest.fixture(scope="session")
def redis_client():
    host = os.getenv("REDIS_HOST", "localhost")
    port = int(os.getenv("REDIS_PORT", "6379"))
    client = redis.Redis(host=host, port=port, decode_responses=True)
    # Verify connection
    client.ping()
    yield client

@pytest.fixture(autouse=True)
def clean_db(redis_client):
    redis_client.flushdb()
    yield
    redis_client.flushdb()
```

## Running Redis Cluster in CI

For cluster tests, use the `grokzen/redis-cluster` Docker image:

```yaml
services:
  redis-cluster:
    image: grokzen/redis-cluster:7.0.10
    ports:
      - 7000-7005:7000-7005
    env:
      IP: "0.0.0.0"
```

Connect in tests:

```python
from redis.cluster import RedisCluster

rc = RedisCluster(host="localhost", port=7000, decode_responses=True)
```

## Caching the Redis Image

Speed up CI by caching the pulled Docker image. In GitHub Actions, the service image is cached automatically across runs on self-hosted runners. For hosted runners, rely on Docker Hub's pull speed.

## Summary

Use CI service containers to spin up Redis alongside your test job, with health checks to ensure Redis is ready before tests start. Read connection details from environment variables so the same test code works locally and in CI. For Redis Cluster tests, use a pre-built cluster Docker image to avoid complex setup scripts.
