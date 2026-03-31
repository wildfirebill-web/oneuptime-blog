# How to Use the setParameter and getParameter Commands in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Configuration, Administration, Parameter, Tuning, Server

Description: Learn how to use setParameter and getParameter in MongoDB to read and modify server configuration at runtime without restarting the process.

---

## Introduction

The `setParameter` and `getParameter` commands allow you to read and modify MongoDB server parameters at runtime. This is useful for tuning behavior without restarting mongod - for example, adjusting log verbosity, changing slow query thresholds, or modifying connection timeouts.

## Getting a Parameter

Retrieve a specific parameter value:

```javascript
db.adminCommand({ getParameter: 1, logLevel: 1 });
// { logLevel: 0, ok: 1 }

db.adminCommand({ getParameter: 1, slowOpThresholdMs: 1 });
// { slowOpThresholdMs: 100, ok: 1 }
```

List all available parameters:

```javascript
db.adminCommand({ getParameter: "*" });
```

## Setting a Parameter

Change a parameter value at runtime:

```javascript
db.adminCommand({ setParameter: 1, logLevel: 2 });
```

Increase the slow operation threshold to reduce log noise:

```javascript
db.adminCommand({ setParameter: 1, slowOpThresholdMs: 500 });
```

Enable query profiling for operations above 200ms:

```javascript
db.setProfilingLevel(1, { slowms: 200 });
```

## Useful Parameters to Know

### Slow Operation Threshold

```javascript
// Get current threshold
db.adminCommand({ getParameter: 1, slowOpThresholdMs: 1 });

// Adjust threshold
db.adminCommand({ setParameter: 1, slowOpThresholdMs: 250 });
```

### Connection Pool Settings

```javascript
db.adminCommand({ getParameter: 1, maxIncomingConnections: 1 });
```

### Failpoint Injection (Testing Only)

```javascript
db.adminCommand({
  configureFailPoint: "failCommand",
  mode: { times: 1 },
  data: { failCommands: ["find"], errorCode: 100 }
});
```

## Verifying a Change

Always confirm parameter changes took effect:

```javascript
function setAndVerify(param, value) {
  db.adminCommand({ setParameter: 1, [param]: value });
  const result = db.adminCommand({ getParameter: 1, [param]: 1 });
  const actual = result[param];
  if (actual !== value) {
    print(`WARNING: expected ${value}, got ${actual}`);
  } else {
    print(`OK: ${param} = ${actual}`);
  }
}

setAndVerify("slowOpThresholdMs", 300);
```

## Persistence Across Restarts

`setParameter` changes are not persisted across restarts. To make them permanent, add them to `mongod.conf`:

```yaml
setParameter:
  slowOpThresholdMs: 300
  logLevel: 0
```

## Summary

The `setParameter` and `getParameter` commands give operators runtime control over MongoDB server behavior without downtime. They are ideal for temporary tuning adjustments, debugging sessions, and emergency configuration changes. For permanent changes, always update `mongod.conf` so settings survive a restart.
