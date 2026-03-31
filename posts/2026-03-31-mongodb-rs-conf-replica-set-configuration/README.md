# How to Use rs.conf() to View Replica Set Configuration in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replica Set, Administration, Configuration, Database

Description: Learn how to use rs.conf() to inspect and understand MongoDB replica set configuration, including members, settings, and version tracking.

---

`rs.conf()` is the primary command for viewing the complete configuration of a MongoDB replica set. Understanding its output is essential for diagnosing replication issues, planning reconfigurations, and verifying member settings.

## Basic Usage

Connect to any replica set member (primary or secondary) and run:

```javascript
rs.conf()
```

This is equivalent to:

```javascript
db.adminCommand({ replSetGetConfig: 1 })
```

## Sample Output

```javascript
{
  _id: "rs0",
  version: 5,
  term: 3,
  protocolVersion: Long("1"),
  writeConcernMajorityJournalDefault: true,
  members: [
    {
      _id: 0,
      host: "mongo1:27017",
      arbiterOnly: false,
      buildIndexes: true,
      hidden: false,
      priority: 1,
      tags: {},
      secondaryDelaySecs: Long("0"),
      votes: 1
    },
    {
      _id: 1,
      host: "mongo2:27017",
      arbiterOnly: false,
      buildIndexes: true,
      hidden: false,
      priority: 1,
      tags: {},
      secondaryDelaySecs: Long("0"),
      votes: 1
    },
    {
      _id: 2,
      host: "mongo3:27017",
      arbiterOnly: true,
      buildIndexes: false,
      hidden: false,
      priority: 0,
      tags: {},
      secondaryDelaySecs: Long("0"),
      votes: 1
    }
  ],
  settings: {
    chainingAllowed: true,
    heartbeatIntervalMillis: 2000,
    heartbeatTimeoutSecs: 10,
    electionTimeoutMillis: 10000,
    catchUpTimeoutMillis: -1,
    getLastErrorModes: {},
    getLastErrorDefaults: { w: 1, wtimeout: 0 },
    replicaSetId: ObjectId("...")
  }
}
```

## Key Fields Explained

| Field | Description |
|---|---|
| `_id` | Replica set name (must match `replSetName` in mongod.conf) |
| `version` | Incremented on every `rs.reconfig()` call |
| `term` | Election term, incremented on each primary election |
| `members[].priority` | Election preference (0 = never primary) |
| `members[].hidden` | Hidden members are invisible to clients |
| `members[].arbiterOnly` | Arbiter members vote but hold no data |
| `members[].secondaryDelaySecs` | Delay before applying oplog (for delayed members) |
| `members[].votes` | 1 = participates in elections, 0 = non-voting |
| `settings.chainingAllowed` | Allow secondaries to sync from other secondaries |
| `settings.electionTimeoutMillis` | Time before starting election after primary loss |

## Accessing Specific Fields

```javascript
// List all member hostnames
rs.conf().members.map(m => m.host)

// Get current version number
rs.conf().version

// Check if chaining is enabled
rs.conf().settings.chainingAllowed

// Find members with priority 0
rs.conf().members.filter(m => m.priority === 0)
```

## Comparing Config Versions

The `version` field helps you verify that a reconfig propagated:

```javascript
// After rs.reconfig(), confirm version incremented
var before = rs.conf().version;
// ... make change ...
var after = rs.conf().version;
print("Version changed: " + (after === before + 1));
```

## Using rs.conf() Before Reconfiguring

Always capture the config before making changes:

```javascript
var cfg = rs.conf();
cfg.members[1].priority = 5;
cfg.settings.chainingAllowed = false;
rs.reconfig(cfg);
```

Modifying the live object and passing it back ensures you don't accidentally reset fields you did not intend to change.

## Summary

`rs.conf()` provides a complete snapshot of your replica set configuration including member roles, priorities, votes, and global settings. Always read it before applying `rs.reconfig()` to avoid unintended changes. The `version` field acts as a change counter to confirm reconfigurations propagated correctly. Use field accessors to query specific properties without parsing the full output manually.

