# How to Optimize ClickHouse Network Settings for WAN

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, WAN, Networking, Performance, Configuration

Description: Learn how to optimize ClickHouse network settings for wide-area network connections to improve performance across high-latency, cross-datacenter links.

---

## WAN Challenges for ClickHouse

ClickHouse is typically deployed on a LAN, but multi-datacenter setups involve WAN connections with higher latency (10-100ms vs. under 1ms on LAN) and limited bandwidth. Without tuning, these conditions cause slow query execution, replication lag, and connection timeouts.

## Increase TCP Buffer Sizes

Larger TCP buffers are essential for high-bandwidth WAN links:

```bash
# Set TCP read and write buffer limits
sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.wmem_max=134217728
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"
```

## Enable TCP BBR Congestion Control

BBR performs significantly better on high-latency links than the default Cubic algorithm:

```bash
sudo sysctl -w net.core.default_qdisc=fq
sudo sysctl -w net.ipv4.tcp_congestion_control=bbr

# Verify
sysctl net.ipv4.tcp_congestion_control
```

## ClickHouse Compression for WAN

Enable compression to reduce the amount of data transferred over WAN:

```xml
<compression>
  <case>
    <method>lz4</method>
  </case>
</compression>
```

For cross-datacenter replication, also set network compression in your client:

```python
client = Client(
    host='remote-ch-host',
    compression=True  # Uses LZ4 compression on the wire
)
```

## Adjust Connection Timeouts for WAN Latency

Increase timeouts to accommodate higher round-trip times:

```xml
<connect_timeout_with_failover_ms>1000</connect_timeout_with_failover_ms>
<receive_timeout>600</receive_timeout>
<send_timeout>600</send_timeout>
```

Default timeouts are tuned for LAN and will cause false failures on WAN links.

## Limit Parallel Cross-Shard Queries

Too many parallel cross-datacenter connections can saturate WAN bandwidth:

```sql
SET max_distributed_connections = 20;
SET distributed_connections_pool_size = 50;
```

## Replication Network Throttling

Prevent replication from saturating your WAN link at the expense of query traffic:

```xml
<max_replicated_fetches_network_bandwidth_for_server>52428800</max_replicated_fetches_network_bandwidth_for_server>
<max_replicated_sends_network_bandwidth_for_server>52428800</max_replicated_sends_network_bandwidth_for_server>
```

These values limit replication to 50 MB/s.

## Monitoring WAN Link Utilization

```bash
# Monitor interface utilization
sar -n DEV 5 10

# Check ClickHouse network metrics
clickhouse-client --query "
  SELECT
    metric,
    value
  FROM system.metrics
  WHERE metric LIKE '%Network%'
"
```

## Summary

Optimizing ClickHouse for WAN requires increasing TCP buffer sizes, switching to BBR congestion control, enabling wire compression, and relaxing timeout values. Throttle replication bandwidth to prevent cross-datacenter data sync from crowding out query traffic. Monitor bandwidth utilization with `sar` and ClickHouse system metrics to identify bottlenecks.
