# How to Use Redis in Jenkins Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Jenkins, CI/CD, Pipeline, Testing

Description: Learn how to run Redis during Jenkins pipeline builds using Docker agents and sidecar containers for integration testing and service dependencies.

---

Jenkins pipelines can use Docker to spin up Redis alongside your build environment. This lets you run integration tests against a real Redis instance without maintaining a dedicated test server.

## Method 1: Docker Agent with Redis Sidecar

Using Jenkins declarative pipeline with Docker agents, you can run Redis as a sidecar:

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '--network=host'
        }
    }
    environment {
        REDIS_HOST = 'localhost'
        REDIS_PORT = '6379'
    }
    stages {
        stage('Start Redis') {
            steps {
                sh 'docker run -d --name redis-test -p 6379:6379 redis:7.2-alpine'
                sh 'sleep 2'
                sh 'docker exec redis-test redis-cli ping'
            }
        }
        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
    }
    post {
        always {
            sh 'docker rm -f redis-test || true'
        }
    }
}
```

## Method 2: Docker Compose in Jenkins

For more complex setups, use Docker Compose:

```groovy
pipeline {
    agent any
    stages {
        stage('Setup Services') {
            steps {
                sh 'docker compose -f docker-compose.test.yml up -d redis'
                sh '''
                    until docker compose -f docker-compose.test.yml exec -T redis redis-cli ping 2>/dev/null; do
                        echo "Waiting for Redis..."
                        sleep 2
                    done
                '''
            }
        }
        stage('Test') {
            steps {
                sh 'docker compose -f docker-compose.test.yml run --rm app npm test'
            }
        }
    }
    post {
        always {
            sh 'docker compose -f docker-compose.test.yml down -v'
        }
    }
}
```

The `docker-compose.test.yml`:

```yaml
services:
  redis:
    image: redis:7.2-alpine
    ports:
    - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  app:
    build: .
    depends_on:
      redis:
        condition: service_healthy
    environment:
      REDIS_URL: redis://redis:6379
```

## Method 3: Kubernetes Agent with Redis Pod

If Jenkins runs on Kubernetes with the Kubernetes Plugin, define Redis as a container in the pod spec:

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: node
    image: node:20-alpine
    command: ["cat"]
    tty: true
  - name: redis
    image: redis:7.2-alpine
    ports:
    - containerPort: 6379
'''
        }
    }
    stages {
        stage('Test') {
            steps {
                container('node') {
                    sh 'npm ci'
                    sh 'npm test'
                    // Redis is available at localhost:6379
                }
            }
        }
    }
}
```

## Python Test Example

```groovy
stage('Python Tests') {
    steps {
        sh '''
            pip install redis pytest
            REDIS_HOST=localhost REDIS_PORT=6379 pytest tests/integration/ -v
        '''
    }
}
```

The test file:

```python
import redis
import os

def test_rate_limiter():
    r = redis.Redis(host=os.getenv("REDIS_HOST", "localhost"),
                    decode_responses=True)
    pipe = r.pipeline()
    pipe.incr("rate:user:1")
    pipe.expire("rate:user:1", 60)
    pipe.execute()
    count = r.get("rate:user:1")
    assert int(count) >= 1
```

## Summary

Jenkins pipelines can integrate Redis through three patterns: Docker sidecar via `--network=host`, Docker Compose for multi-service tests, or Kubernetes pod containers when Jenkins runs in Kubernetes. Always clean up the Redis container in a `post { always }` block to avoid resource leaks. Use a readiness check before running tests to prevent flaky failures.
