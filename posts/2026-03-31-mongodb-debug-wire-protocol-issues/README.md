# How to Debug MongoDB Wire Protocol Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Wire Protocol, Debugging, Troubleshooting, Networking

Description: Debug MongoDB wire protocol issues using packet capture, driver logging, mongosh tracing, and mongosniff to diagnose communication problems between clients and servers.

---

## When Wire Protocol Debugging is Needed

Wire protocol debugging is necessary when:
- Driver errors are vague and don't map to a specific MongoDB command
- A proxy or load balancer is silently modifying or dropping packets
- You suspect a driver version incompatibility with the server
- Performance profiling needs to identify expensive commands at the network level

## Tool 1: Enable Driver Command Monitoring

Most drivers emit command-level events before resorting to packet capture. Enable monitoring in Node.js:

```javascript
const client = new MongoClient(uri, {
  monitorCommands: true
});

client.on("commandStarted", (event) => {
  console.log("Command started:", JSON.stringify({
    command: event.commandName,
    db: event.databaseName,
    requestId: event.requestId,
    body: event.command
  }, null, 2));
});

client.on("commandSucceeded", (event) => {
  console.log("Command succeeded:", event.commandName, `${event.duration}ms`);
});

client.on("commandFailed", (event) => {
  console.error("Command failed:", event.commandName, event.failure);
});
```

In Python with PyMongo:

```python
from pymongo import monitoring

class CommandLogger(monitoring.CommandListener):
    def started(self, event):
        print(f"Started {event.command_name} (id={event.request_id})")
    def succeeded(self, event):
        print(f"Succeeded {event.command_name} in {event.duration_micros}us")
    def failed(self, event):
        print(f"Failed {event.command_name}: {event.failure}")

monitoring.register(CommandLogger())
```

## Tool 2: Capture Traffic with tcpdump

Capture raw MongoDB packets to a file for analysis:

```bash
sudo tcpdump -i any -w /tmp/mongo_debug.pcap 'tcp port 27017'
```

Capture with readable ASCII output for quick inspection:

```bash
sudo tcpdump -i any -A 'tcp port 27017' | grep -A5 "find\|insert\|update\|delete"
```

## Tool 3: Analyze with Wireshark

Open the pcap file in Wireshark. Filter for MongoDB traffic:

```text
mongo
```

Wireshark's MongoDB dissector decodes OP_MSG sections and shows:
- Command names
- Database and collection
- Query predicates
- Response documents
- Round-trip time per request

## Tool 4: Use mongosniff

`mongosniff` is a MongoDB-specific packet sniffer included with older MongoDB distributions:

```bash
sudo mongosniff --source NET lo0
```

For a file:

```bash
sudo mongosniff --source FILE /tmp/mongo_debug.pcap
```

## Tool 5: Enable mongod Diagnostic Logging

Increase verbosity for network-level logging:

```javascript
db.adminCommand({
  setParameter: 1,
  logComponentVerbosity: {
    network: { verbosity: 2 },
    command: { verbosity: 1 }
  }
})
```

View the log:

```bash
sudo tail -f /var/log/mongodb/mongod.log | grep '"c":"NETWORK"'
```

## Common Wire Protocol Issues and Causes

```text
Issue: "message length too long"
Cause: Sending documents exceeding 16MB BSON limit

Issue: "bad wire protocol message opCode"
Cause: Driver using legacy opcodes against MongoDB 5.0+ with legacy opcode support disabled

Issue: "SSL handshake failed"
Cause: TLS version mismatch or certificate validation failure at the transport layer

Issue: Timeout with no server error
Cause: TCP connection silently dropped by a firewall or NAT timeout
```

## Summary

MongoDB wire protocol debugging starts with driver-level command monitoring events, which provide command names, request IDs, and durations without requiring network access. For deeper inspection, use `tcpdump` to capture raw packets and Wireshark to decode OP_MSG frames. Increase `mongod` log verbosity for network and command components to see server-side perspectives on connection events. This multi-layer approach isolates whether issues originate in the driver, network layer, or server.
