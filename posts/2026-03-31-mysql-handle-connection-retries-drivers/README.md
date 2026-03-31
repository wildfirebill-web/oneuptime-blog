# How to Handle Connection Retries in MySQL Drivers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Connection, Retry, Resilience, Driver

Description: Learn how to implement connection retry logic in MySQL drivers using exponential backoff to build resilient applications that recover from transient failures.

---

## Overview

Transient connection failures are inevitable in distributed systems - network blips, server restarts, and failovers all cause temporary unavailability. Implementing retry logic with exponential backoff ensures your application recovers gracefully without overwhelming the database during recovery.

## When to Retry vs. When to Fail Fast

Not all errors are retriable. Retry on transient errors, fail fast on permanent ones:

```text
Retriable errors:
  - 2003: Can't connect to MySQL server (ECONNREFUSED)
  - 2006: MySQL server has gone away
  - 2013: Lost connection to MySQL server during query
  - 1040: Too many connections

Non-retriable errors:
  - 1045: Access denied (wrong credentials)
  - 1049: Unknown database
  - 1142: Permission denied on table
```

## Implementing Retry with Exponential Backoff in Python

```python
import mysql.connector
import time
import random

RETRIABLE_ERRORS = {2003, 2006, 2013, 1040}

def connect_with_retry(config, max_attempts=5, base_delay=1.0):
    """Connect to MySQL with exponential backoff retry."""
    attempt = 0
    while attempt < max_attempts:
        try:
            conn = mysql.connector.connect(**config)
            print(f"Connected after {attempt + 1} attempt(s)")
            return conn
        except mysql.connector.Error as e:
            if e.errno not in RETRIABLE_ERRORS:
                raise  # Don't retry permanent errors

            attempt += 1
            if attempt >= max_attempts:
                raise

            # Exponential backoff with jitter
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            print(f"Connection failed (attempt {attempt}): {e.msg}. "
                  f"Retrying in {delay:.1f}s...")
            time.sleep(delay)

config = {
    'host': 'localhost',
    'user': 'app_user',
    'password': 'secure_password',
    'database': 'myapp'
}

conn = connect_with_retry(config)
```

## Implementing Retry in Node.js

```javascript
const mysql = require('mysql2/promise');

const RETRIABLE_CODES = ['ECONNREFUSED', 'PROTOCOL_CONNECTION_LOST',
                          'ECONNRESET', 'ER_CON_COUNT_ERROR'];

async function connectWithRetry(config, maxAttempts = 5) {
  let attempt = 0;
  while (attempt < maxAttempts) {
    try {
      const connection = await mysql.createConnection(config);
      console.log(`Connected after ${attempt + 1} attempt(s)`);
      return connection;
    } catch (err) {
      if (!RETRIABLE_CODES.includes(err.code)) {
        throw err; // Permanent error, do not retry
      }

      attempt++;
      if (attempt >= maxAttempts) throw err;

      const delay = Math.pow(2, attempt) * 1000 + Math.random() * 1000;
      console.log(`Connection failed: ${err.message}. Retrying in ${(delay/1000).toFixed(1)}s...`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

const config = {
  host: 'localhost',
  user: 'app_user',
  password: 'secure_password',
  database: 'myapp'
};

connectWithRetry(config).then(conn => {
  // use connection
});
```

## Retry Logic for Query Execution

Beyond initial connection, queries can also fail mid-execution during failovers:

```python
def execute_with_retry(conn, query, params=None, max_attempts=3):
    """Execute a query with retry on connection loss."""
    for attempt in range(max_attempts):
        try:
            cursor = conn.cursor(dictionary=True)
            cursor.execute(query, params or [])
            return cursor.fetchall()
        except mysql.connector.Error as e:
            if e.errno in {2006, 2013} and attempt < max_attempts - 1:
                print(f"Lost connection, reconnecting (attempt {attempt + 1})...")
                conn.reconnect(attempts=3, delay=2)
                continue
            raise
```

## Circuit Breaker Pattern

For high-traffic systems, combine retry logic with a circuit breaker:

```python
from datetime import datetime, timedelta

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN

    def can_attempt(self):
        if self.state == 'CLOSED':
            return True
        if self.state == 'OPEN':
            if datetime.now() - self.last_failure_time > timedelta(seconds=self.recovery_timeout):
                self.state = 'HALF_OPEN'
                return True
            return False
        return True  # HALF_OPEN: allow one attempt

    def record_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
```

## Summary

Effective MySQL connection retry logic combines distinguishing retriable from non-retriable errors, exponential backoff with jitter to spread load, maximum attempt limits to avoid infinite loops, and circuit breakers to prevent retry storms during extended outages. Always implement jitter (randomness) in your backoff delay to prevent thundering herd problems when many clients simultaneously try to reconnect after a database restart.
