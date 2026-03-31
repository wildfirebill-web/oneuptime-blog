# How to Set Up the Telegraf Module in Ceph Manager

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Telegraf, Monitoring, InfluxDB

Description: Learn how to enable and configure the Ceph Manager Telegraf module to send cluster metrics to a Telegraf agent for flexible metrics routing.

---

The Ceph Manager Telegraf module sends cluster performance statistics to a Telegraf agent using the socket listener plugin. This integrates Ceph into the Telegraf ecosystem, allowing you to route metrics to InfluxDB, Prometheus, Elasticsearch, or any other Telegraf output plugin.

## Enabling the Module

Load the Telegraf module in the Ceph Manager:

```bash
ceph mgr module enable telegraf
```

Confirm it is active:

```bash
ceph mgr module ls | grep telegraf
```

## Configuring the Telegraf Socket

The module communicates with Telegraf over a Unix socket or TCP connection. Set the address:

```bash
# Unix socket
ceph config set mgr mgr/telegraf/address unixgram:///tmp/ceph-telegraf.sock

# TCP socket
ceph config set mgr mgr/telegraf/address tcp://127.0.0.1:8094
```

Set the interval between metric pushes (seconds):

```bash
ceph config set mgr mgr/telegraf/interval 15
```

## Configuring Telegraf to Receive Data

Add a socket listener input to your `telegraf.conf`:

```toml
[[inputs.socket_listener]]
  service_address = "unixgram:///tmp/ceph-telegraf.sock"
  data_format = "influx"

[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "my-token"
  organization = "my-org"
  bucket = "ceph"
```

Restart Telegraf after updating the config:

```bash
sudo systemctl restart telegraf
```

## Verifying Metric Flow

Check that the socket is created and data is flowing:

```bash
ls -la /tmp/ceph-telegraf.sock
```

Enable Telegraf debug logging to inspect incoming metrics:

```toml
[agent]
  debug = true
  logfile = "/var/log/telegraf/telegraf.log"
```

```bash
tail -f /var/log/telegraf/telegraf.log | grep ceph
```

## Example Metrics Sent

The Telegraf module pushes measurements including:

- `ceph_osd` - OSD performance stats per daemon
- `ceph_pool` - Per-pool IOPS and bandwidth
- `ceph_health` - Overall cluster health status
- `ceph_mon` - Monitor quorum and latency metrics

## Routing to Multiple Outputs

One advantage of routing through Telegraf is multi-destination output:

```toml
[[outputs.prometheus_client]]
  listen = ":9273"

[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "token"
  organization = "org"
  bucket = "ceph"
```

## Summary

The Ceph Manager Telegraf module pushes metrics to a Telegraf socket, integrating Ceph into a flexible metrics pipeline. By configuring the socket address and interval in Ceph, and the socket listener in Telegraf, you can route Ceph metrics to any backend Telegraf supports, including InfluxDB, Prometheus, and Elasticsearch.
