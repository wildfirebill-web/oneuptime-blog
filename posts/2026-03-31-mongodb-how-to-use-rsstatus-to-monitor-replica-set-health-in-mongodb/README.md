# How to Use rs.status() to Monitor Replica Set Health in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replica Set, Monitoring, Operations, Health Check

Description: Use rs.status() and related commands to monitor MongoDB replica set health, replication lag, and member states in real time.

---

## Running rs.status()

The `rs.status()` command returns the current health and state of every replica set member. Run it from any member using `mongosh`:

```javascript
mongosh --host any-member:27017
rs.status()
```

## Key Fields in rs.status() Output

The output is a document with a `members` array. The most important fields per member are:

```javascript
{
  name: "192.168.1.10:27017",
  health: 1,               // 1=healthy, 0=unhealthy
  state: 1,                // 1=PRIMARY, 2=SECONDARY, 6=UNKNOWN, 8=DOWN
  stateStr: "PRIMARY",
  uptime: 86400,
  optime: { ts: Timestamp(...), t: 5 },
  optimeDate: ISODate("2026-03-31T10:00:00Z"),
  lastHeartbeatMessage: "",
  syncSourceHost: "",
  configVersion: 3
}
```

## Checking for Unhealthy Members

Quickly filter for members that are not in a healthy state:

```javascript
rs.status().members.filter(m => m.health !== 1 || m.state > 2)
```

State codes to watch for:

- `6` - UNKNOWN (recently lost contact)
- `8` - DOWN (unreachable)
- `9` - ROLLBACK (rolling back writes after re-joining)
- `10` - REMOVED (removed from config)

## Measuring Replication Lag

Compare `optimeDate` of the primary to each secondary:

```javascript
const status = rs.status();
const primary = status.members.find(m => m.stateStr === "PRIMARY");

status.members
  .filter(m => m.stateStr === "SECONDARY")
  .forEach(sec => {
    const lagMs = primary.optimeDate - sec.optimeDate;
    print(`${sec.name}: lag = ${lagMs / 1000}s`);
  });
```

Or use the convenience command:

```javascript
rs.printSecondaryReplicationInfo()
```

## Interpreting optime and lastHeartbeat

The `optime` reflects the last oplog entry applied. High lag indicates the secondary is falling behind writes. Check `lastHeartbeatMessage` for error messages when a member is unreachable.

```javascript
rs.status().members.forEach(m => {
  if (m.lastHeartbeatMessage) {
    print(`${m.name}: ${m.lastHeartbeatMessage}`);
  }
});
```

## Automating Health Checks

Script a basic health check to alert on issues:

```javascript
const status = rs.status();
const issues = status.members.filter(
  m => m.health !== 1 || !["PRIMARY", "SECONDARY", "ARBITER"].includes(m.stateStr)
);

if (issues.length > 0) {
  print("ALERT: Replica set has unhealthy members:");
  printjson(issues.map(m => ({ name: m.name, state: m.stateStr, health: m.health })));
} else {
  print("Replica set is healthy");
}
```

## Summary

`rs.status()` is the primary tool for monitoring MongoDB replica set health. Check the `health` field and `stateStr` for each member, measure replication lag by comparing `optimeDate` values, and automate checks by scripting `mongosh` commands. Alert on any member with `health: 0` or unexpected state codes like DOWN or ROLLBACK.
