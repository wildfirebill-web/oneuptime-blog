# How to Configure NATS Server with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, NATS, Message Queues, Pub/Sub, Messaging, Cloud Native

Description: Learn how to configure NATS Server to listen on IPv6 addresses for client connections, cluster routes, and monitoring, enabling IPv6-native messaging deployments.

## NATS Server Configuration File

```conf
# /etc/nats/nats-server.conf

# Server name

server_name: nats-node1

# Listen on specific IPv6 address for client connections
host: "2001:db8::10"
port: 4222

# Or listen on all interfaces
# host: "::"
# port: 4222

# HTTP monitoring
http: "[2001:db8::10]:8222"

# Cluster configuration
cluster {
  name: my-cluster
  host: "2001:db8::10"
  port: 6222
  routes: [
    "nats://[2001:db8::11]:6222"
    "nats://[2001:db8::12]:6222"
  ]
}
```

## Start NATS with IPv6 via CLI Flags

```bash
# Start NATS on specific IPv6 address
nats-server --addr 2001:db8::10 --port 4222

# Listen on all IPv6 interfaces
nats-server --addr :: --port 4222

# With monitoring
nats-server --addr 2001:db8::10 --port 4222 --http_port 8222

# Using config file
nats-server -c /etc/nats/nats-server.conf
```

## Verify and Test

```bash
# Restart NATS service
systemctl restart nats-server

# Check listening ports
ss -6 -tlnp | grep nats
# Expected: [2001:db8::10]:4222, [2001:db8::10]:8222

# Check monitoring endpoint
curl -6 http://[2001:db8::10]:8222/varz

# Check server info via NATS
nats server info --server nats://[2001:db8::10]:4222

# Subscribe to a subject
nats sub --server nats://[2001:db8::10]:4222 "test.subject" &

# Publish a message
nats pub --server nats://[2001:db8::10]:4222 "test.subject" "Hello IPv6!"
```

## NATS JetStream with IPv6

```conf
# /etc/nats/nats-server.conf with JetStream

host: "2001:db8::10"
port: 4222

# Enable JetStream
jetstream {
  store_dir: /var/lib/nats/jetstream
  max_memory_store: 1GB
  max_file_store: 10GB
}

# Monitoring
http: "[2001:db8::10]:8222"
```

## Go NATS Client over IPv6

```go
package main

import (
    "fmt"
    "time"
    nats "github.com/nats-io/nats.go"
)

func main() {
    // Connect to NATS via IPv6
    nc, err := nats.Connect("nats://[2001:db8::10]:4222",
        nats.Timeout(5*time.Second),
        nats.RetryOnFailedConnect(true),
    )
    if err != nil {
        panic(err)
    }
    defer nc.Close()

    // Subscribe to a subject
    sub, err := nc.Subscribe("updates", func(msg *nats.Msg) {
        fmt.Printf("Received: %s\n", string(msg.Data))
    })
    if err != nil {
        panic(err)
    }
    defer sub.Unsubscribe()

    // Publish a message
    err = nc.Publish("updates", []byte("Hello from IPv6 NATS!"))
    if err != nil {
        panic(err)
    }

    nc.Flush()
    time.Sleep(100 * time.Millisecond)
}
```

## Python NATS Client over IPv6

```python
import asyncio
import nats

async def main():
    # Connect to NATS via IPv6
    nc = await nats.connect("nats://[2001:db8::10]:4222")

    async def message_handler(msg):
        print(f"Received: {msg.data.decode()}")

    # Subscribe
    sub = await nc.subscribe("test.subject", cb=message_handler)

    # Publish
    await nc.publish("test.subject", b"Hello IPv6!")

    await asyncio.sleep(1)
    await sub.unsubscribe()
    await nc.close()

asyncio.run(main())
```

## Summary

Configure NATS Server for IPv6 by setting `host: "2001:db8::10"` in the configuration file or using `--addr 2001:db8::10` CLI flag. For all interfaces, use `host: "::"`. Configure cluster routes with bracket notation: `nats://[2001:db8::11]:6222`. Monitor with `curl -6 http://[2001:db8::10]:8222/varz`. Connect clients using `nats://[2001:db8::10]:4222`. Restart with `systemctl restart nats-server`.
