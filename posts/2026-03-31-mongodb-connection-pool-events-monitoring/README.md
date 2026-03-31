# How to Use Connection Pool Events for Monitoring in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Connection Pool, CMAP, Monitoring, Observability

Description: Use MongoDB CMAP specification events to instrument connection pool lifecycle, detect anomalies, and build real-time dashboards for pool health monitoring.

---

## What are CMAP Events

The Connection Monitoring and Pooling (CMAP) specification defines a standard set of events that MongoDB drivers emit throughout a connection's lifecycle. These events provide deep visibility into pool behavior without requiring packet capture or server-side profiling.

## Complete CMAP Event Reference

```text
connectionPoolCreated      - Pool initialized for a server address
connectionPoolReady        - Pool is ready to serve connections
connectionPoolCleared      - All connections reset (topology change, server failure)
connectionPoolClosed       - Pool shut down (client closing)
connectionCreated          - New TCP connection established
connectionReady            - Connection passed handshake, ready for use
connectionClosed           - Connection closed (idle timeout, error, or shutdown)
connectionCheckOutStarted  - Operation requested a connection from pool
connectionCheckedOut       - Connection successfully given to an operation
connectionCheckOutFailed   - Could not obtain a connection (timeout or pool exhausted)
connectionCheckedIn        - Connection returned to pool after operation
```

## Full CMAP Instrumentation in Node.js

```javascript
const { MongoClient } = require("mongodb");

class PoolMonitor {
  constructor() {
    this.pools = new Map();
  }

  attach(client) {
    client.on("connectionPoolCreated", ({ address, options }) => {
      this.pools.set(address, {
        address,
        maxSize: options.maxPoolSize,
        total: 0,
        checkedOut: 0,
        waitQueueDepth: 0,
        checkoutFailures: 0
      });
      console.log(`Pool created for ${address} (max: ${options.maxPoolSize})`);
    });

    client.on("connectionCreated", ({ address }) => {
      const pool = this.pools.get(address);
      if (pool) pool.total++;
    });

    client.on("connectionClosed", ({ address, reason }) => {
      const pool = this.pools.get(address);
      if (pool) {
        pool.total--;
        if (reason === "error") {
          console.warn(`Connection closed due to error on ${address}`);
        }
      }
    });

    client.on("connectionCheckedOut", ({ address }) => {
      const pool = this.pools.get(address);
      if (pool) pool.checkedOut++;
    });

    client.on("connectionCheckedIn", ({ address }) => {
      const pool = this.pools.get(address);
      if (pool) pool.checkedOut--;
    });

    client.on("connectionCheckOutFailed", ({ address, reason }) => {
      const pool = this.pools.get(address);
      if (pool) {
        pool.checkoutFailures++;
        console.error(`Pool checkout failed on ${address}: ${reason}`);
      }
    });

    client.on("connectionPoolCleared", ({ address, serviceId }) => {
      console.warn(`Pool cleared for ${address} - all connections being reset`);
    });
  }

  getUtilization(address) {
    const pool = this.pools.get(address);
    if (!pool || pool.maxSize === 0) return 0;
    return pool.checkedOut / pool.maxSize;
  }

  report() {
    for (const [address, pool] of this.pools) {
      const utilPct = (this.getUtilization(address) * 100).toFixed(1);
      console.log(
        `[${address}] total=${pool.total} checkedOut=${pool.checkedOut} ` +
        `utilization=${utilPct}% failures=${pool.checkoutFailures}`
      );
    }
  }
}

const monitor = new PoolMonitor();
const client = new MongoClient(uri, { maxPoolSize: 50 });
monitor.attach(client);

// Report metrics every 10 seconds
setInterval(() => monitor.report(), 10000);
```

## Alerting on Pool Anomalies

```javascript
client.on("connectionCheckOutFailed", ({ address, reason }) => {
  if (reason === "timeout") {
    alerting.trigger("mongodb_pool_timeout", {
      severity: "warning",
      message: `Connection pool timeout on ${address}`,
      runbook: "https://runbooks.example.com/mongodb-pool"
    });
  }
});

client.on("connectionPoolCleared", ({ address }) => {
  alerting.trigger("mongodb_pool_cleared", {
    severity: "critical",
    message: `Pool cleared unexpectedly for ${address}`
  });
});
```

## Exporting CMAP Metrics via OpenTelemetry

```javascript
const { metrics } = require("@opentelemetry/api");
const meter = metrics.getMeter("mongodb-pool");

const checkedOutGauge = meter.createObservableGauge("mongodb.pool.checked_out");
const utilizationGauge = meter.createObservableGauge("mongodb.pool.utilization");

meter.addBatchObservableCallback((observableResult) => {
  for (const [address, pool] of monitor.pools) {
    const attrs = { "db.server.address": address };
    observableResult.observe(checkedOutGauge, pool.checkedOut, attrs);
    observableResult.observe(utilizationGauge, monitor.getUtilization(address), attrs);
  }
}, [checkedOutGauge, utilizationGauge]);
```

## Summary

MongoDB CMAP events provide a complete picture of connection pool health including creation, checkout, check-in, failures, and pool resets. Attach event listeners at client initialization to track utilization (`checkedOut / maxPoolSize`), checkout failure rates, and pool clear events (which indicate server failures or topology changes). Export these metrics to your observability platform and alert when utilization exceeds 80% or checkout failures begin occurring, indicating the pool needs tuning.
