# How to Configure gRPC Keepalive for IPv4 Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: gRPC, Keepalive, IPv4, Python, Go, Networking

Description: Learn how to configure gRPC keepalive parameters to detect dead IPv4 connections, prevent firewall idle timeouts, and ensure long-lived RPC streams remain healthy.

## Why Keepalive Matters

Firewalls and NAT devices silently drop idle TCP connections after a timeout (typically 60–300 seconds). gRPC uses HTTP/2 PING frames as a keepalive mechanism to probe the connection without sending application data.

```text
Client                        Server
  │                               │
  │──── HTTP/2 PING ─────────────►│
  │◄─── PING ACK ─────────────────│
  │        (connection alive)     │
  │                               │
  │  (no data for ping_interval)  │
  │──── HTTP/2 PING ─────────────►│
  │  (no reply within timeout) X  │
  │  → close & reconnect          │
```

## Python: Client Keepalive

```python
import grpc

channel = grpc.insecure_channel(
    "192.168.1.10:50051",
    options=[
        # Send a keepalive ping every 30 seconds of inactivity
        ("grpc.keepalive_time_ms", 30_000),
        # Wait 10 seconds for ping ack before declaring connection dead
        ("grpc.keepalive_timeout_ms", 10_000),
        # Send keepalive even when there are no active RPCs
        ("grpc.keepalive_permit_without_calls", True),
        # Allow at most 5 pings before server must send data or close
        ("grpc.http2.max_pings_without_data", 5),
    ],
)
```

## Python: Server Keepalive

```python
import grpc
from concurrent import futures

server = grpc.server(
    futures.ThreadPoolExecutor(max_workers=10),
    options=[
        # Allow clients to send keepalive pings every 10s minimum
        ("grpc.http2.min_ping_interval_without_data_ms", 10_000),
        # Maximum time a client can go without any data/ping
        ("grpc.http2.max_connection_idle_ms", 300_000),  # 5 min
        # Maximum connection age
        ("grpc.http2.max_connection_age_ms", 3_600_000),  # 1 hour
        # Grace period to finish RPCs before closing
        ("grpc.http2.max_connection_age_grace_ms", 30_000),
    ],
)
```

## Go: Client Keepalive

```go
import (
    "time"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/keepalive"
)

conn, err := grpc.NewClient(
    "192.168.1.10:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                30 * time.Second, // ping interval
        Timeout:             10 * time.Second, // ping ack wait
        PermitWithoutStream: true,             // ping even without active RPC
    }),
)
```

## Go: Server Keepalive

```go
import (
    "time"
    "google.golang.org/grpc"
    "google.golang.org/grpc/keepalive"
)

s := grpc.NewServer(
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle:     5 * time.Minute,
        MaxConnectionAge:      1 * time.Hour,
        MaxConnectionAgeGrace: 30 * time.Second,
        Time:                  30 * time.Second,
        Timeout:               10 * time.Second,
    }),
    grpc.KeepaliveEnforcementPolicy(keepalive.EnforcementPolicy{
        MinTime:             10 * time.Second, // min ping interval from clients
        PermitWithoutStream: true,
    }),
)
```

## Keepalive Parameter Reference

| Parameter | Recommended Value | Notes |
|-----------|------------------|-------|
| `keepalive_time_ms` | 30 000 ms | Less than firewall idle timeout |
| `keepalive_timeout_ms` | 10 000 ms | How long to wait for PING ACK |
| `PermitWithoutStream` | `true` | Keep connection alive with no active RPCs |
| `MaxConnectionIdle` | 5 min | Close idle connections gracefully |
| `MaxConnectionAge` | 1 hour | Force reconnect to rebalance |

## Conclusion

Set `keepalive_time_ms` (client) and the server `Time` parameter to a value shorter than your network's firewall idle timeout - 30 seconds is a safe default. Enable `PermitWithoutStream` so the connection stays alive even when there are no active RPCs. Configure `MaxConnectionAge` on the server to periodically force clients to reconnect, which rebalances connections after pod restarts. Mismatched keepalive policies cause `ENHANCE_YOUR_CALM (too_many_pings)` errors - coordinate client and server settings.
