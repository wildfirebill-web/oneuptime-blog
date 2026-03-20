# How to Configure InfluxDB to Listen on IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, InfluxDB, Time Series Database, Monitoring, Metrics

Description: Learn how to configure InfluxDB to listen on IPv6 addresses for HTTP API and cluster communication, enabling IPv6-only and dual-stack deployments.

## InfluxDB HTTP Configuration

```toml
# /etc/influxdb/influxdb.conf (InfluxDB 1.x)

[http]
  # Bind to specific IPv6 address
  bind-address = "[2001:db8::10]:8086"

  # Bind to all interfaces (IPv4 and IPv6)
  # bind-address = "[::]:8086"

  # Bind to IPv6 loopback
  # bind-address = "[::1]:8086"

  enabled = true
  auth-enabled = false
```

## InfluxDB 2.x Configuration

```yaml
# /etc/influxdb/config.yml (InfluxDB 2.x)

# HTTP bind address
http-bind-address: "[2001:db8::10]:8086"

# Or listen on all interfaces
# http-bind-address: "[::]:8086"
```

## Start InfluxDB with IPv6 via CLI Flags

```bash
# InfluxDB 2.x - start with IPv6 bind address
influxd --http-bind-address "[2001:db8::10]:8086"

# Listen on all IPv6 interfaces
influxd --http-bind-address "[::]:8086"

# With storage path
influxd \
    --http-bind-address "[2001:db8::10]:8086" \
    --bolt-path /var/lib/influxdb/influxd.bolt \
    --engine-path /var/lib/influxdb/engine
```

## Verify InfluxDB IPv6 Binding

```bash
# Restart InfluxDB
systemctl restart influxdb

# Check listening ports
ss -6 -tlnp | grep influx
# Expected: [2001:db8::10]:8086

# Test HTTP API over IPv6
curl -6 http://[2001:db8::10]:8086/ping
# Should return: 204 No Content

# InfluxDB 2.x health check
curl -6 http://[2001:db8::10]:8086/health
```

## Write and Query Data over IPv6

```bash
# InfluxDB 1.x - Write data via line protocol
curl -6 -X POST "http://[2001:db8::10]:8086/write?db=mydb" \
    --data-binary "temperature,host=server01 value=23.5 $(date +%s)000000000"

# Query data
curl -6 "http://[2001:db8::10]:8086/query?db=mydb&q=SELECT+*+FROM+temperature"

# InfluxDB 2.x - Write data
curl -6 -X POST "http://[2001:db8::10]:8086/api/v2/write?org=myorg&bucket=mybucket&precision=s" \
    -H "Authorization: Token mytoken" \
    --data-binary "temperature,host=server01 value=23.5"

# InfluxDB 2.x - Query with Flux
curl -6 -X POST "http://[2001:db8::10]:8086/api/v2/query?org=myorg" \
    -H "Authorization: Token mytoken" \
    -H "Content-Type: application/vnd.flux" \
    --data 'from(bucket:"mybucket") |> range(start: -1h) |> filter(fn: (r) => r._measurement == "temperature")'
```

## influx CLI with IPv6

```bash
# Configure influx CLI to connect via IPv6
influx config create \
    --config-name myconfig \
    --host-url "http://[2001:db8::10]:8086" \
    --org myorg \
    --token mytoken \
    --active

# Use the CLI
influx write --bucket mybucket "temperature,host=server01 value=25.0"
influx query 'from(bucket:"mybucket") |> range(start: -1h)'

# List buckets
influx bucket list
```

## Python InfluxDB Client over IPv6

```python
from influxdb_client import InfluxDBClient, WriteOptions

# Connect to InfluxDB via IPv6
client = InfluxDBClient(
    url="http://[2001:db8::10]:8086",
    token="mytoken",
    org="myorg"
)

write_api = client.write_api()
query_api = client.query_api()

# Write a data point
from influxdb_client.client.write_api import SYNCHRONOUS
write_api = client.write_api(write_options=SYNCHRONOUS)
write_api.write(
    bucket="mybucket",
    record="temperature,host=server01 value=23.5"
)

# Query data
result = query_api.query('from(bucket:"mybucket") |> range(start: -1h)')
for table in result:
    for record in table.records:
        print(f"{record.get_time()}: {record.get_value()}")

client.close()
```

## Summary

Configure InfluxDB 1.x for IPv6 with `bind-address = "[2001:db8::10]:8086"` in the `[http]` section of `influxdb.conf`. For InfluxDB 2.x, use `http-bind-address: "[2001:db8::10]:8086"` in `config.yml` or pass `--http-bind-address "[2001:db8::10]:8086"` as a CLI flag. Use `[::]` to listen on all interfaces. Test with `curl -6 http://[2001:db8::10]:8086/ping`. Restart with `systemctl restart influxdb`.
