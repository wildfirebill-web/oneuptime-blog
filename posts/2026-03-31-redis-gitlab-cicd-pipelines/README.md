# How to Use Redis in GitLab CI/CD Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, GitLab, CI/CD, Service, Testing

Description: Learn how to add Redis as a service in GitLab CI/CD pipelines for integration testing, with examples for Python, Node.js, and Ruby applications.

---

GitLab CI/CD supports service containers that run alongside your job container. Adding Redis as a service lets your tests interact with a real Redis instance without any manual setup.

## Basic Redis Service in .gitlab-ci.yml

```yaml
test:
  image: node:20-alpine
  services:
  - name: redis:7.2-alpine
    alias: redis
  variables:
    REDIS_HOST: redis
    REDIS_PORT: "6379"
  script:
  - npm ci
  - npm test
```

The `alias` field sets the hostname your job can use to reach Redis. Without it, the hostname defaults to `redis` for the official Redis image.

## Redis with Password Authentication

```yaml
test:
  image: python:3.12-slim
  services:
  - name: redis:7.2-alpine
    alias: redis
    command: ["redis-server", "--requirepass", "testpassword"]
  variables:
    REDIS_URL: redis://:testpassword@redis:6379
  script:
  - pip install -r requirements.txt pytest
  - pytest tests/integration/ -v
```

## Wait for Redis to be Ready

Redis starts quickly, but adding a readiness check prevents flaky tests:

```yaml
test:
  image: python:3.12-slim
  services:
  - name: redis:7.2-alpine
    alias: redis
  before_script:
  - apt-get update -q && apt-get install -y redis-tools
  - |
    until redis-cli -h redis ping 2>/dev/null; do
      echo "Waiting for Redis..."
      sleep 1
    done
    echo "Redis is ready"
  script:
  - pip install -r requirements.txt pytest
  - pytest tests/
```

## Python Example with Caching Tests

```yaml
test:unit:
  image: python:3.12-slim
  services:
  - name: redis:7.2-alpine
    alias: redis
  variables:
    REDIS_HOST: redis
    REDIS_PORT: "6379"
  script:
  - pip install redis pytest
  - pytest tests/unit/ -v

test:integration:
  image: python:3.12-slim
  services:
  - name: redis:7.2-alpine
    alias: redis
  variables:
    REDIS_HOST: redis
    REDIS_PORT: "6379"
  script:
  - pip install redis pytest
  - pytest tests/integration/ -v
```

Your integration test:

```python
import redis
import os

def get_redis():
    return redis.Redis(
        host=os.getenv("REDIS_HOST", "localhost"),
        port=int(os.getenv("REDIS_PORT", 6379)),
        decode_responses=True
    )

def test_cache_set_and_get():
    r = get_redis()
    r.set("ci_test", "hello", ex=30)
    assert r.get("ci_test") == "hello"
    r.delete("ci_test")
```

## Ruby on Rails with Redis (Sidekiq)

```yaml
test:
  image: ruby:3.3
  services:
  - name: redis:7.2-alpine
    alias: redis
  variables:
    REDIS_URL: redis://redis:6379/0
    RAILS_ENV: test
  script:
  - bundle install
  - bundle exec rails db:schema:load
  - bundle exec rspec
```

## Node.js with BullMQ

```yaml
test:
  image: node:20-alpine
  services:
  - name: redis:7.2-alpine
    alias: redis
  variables:
    REDIS_HOST: redis
    REDIS_PORT: "6379"
  script:
  - npm ci
  - npm run test:integration
```

## Summary

GitLab CI/CD makes Redis available as a named service container that your job container accesses by hostname. Use the `alias` field to give Redis a predictable hostname, add a readiness check in `before_script` to avoid race conditions, and pass connection details via CI/CD variables. This pattern works identically for caching, job queues, and session storage in any language.
