# How to Monitor TCP Statistics and RTT for Ceph Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Network, TCP, Performance

Description: Monitor TCP connection statistics and round-trip time (RTT) for Ceph daemon connections to identify network bottlenecks affecting cluster performance.

---

Ceph is highly sensitive to network latency. High TCP round-trip time (RTT) between OSDs and monitors directly impacts write latency, as Ceph must wait for acknowledgment from multiple replicas before confirming a write to clients.

## Why TCP RTT Matters for Ceph

- OSD replication requires synchronous acknowledgment across `min_size` replicas
- Monitor operations include round-trips for map updates
- High RTT directly adds to client-perceived write latency
- Typical LAN RTT: <1ms; acceptable for Ceph
- RTT >5ms noticeably impacts write performance; >20ms causes significant degradation

## Check TCP Connections from OSD Pods

```bash
# Get the OSD pod name
OSD_POD=$(kubectl get pod -n rook-ceph -l ceph_daemon_type=osd,ceph_daemon_id=0 -o name)

# List TCP connections
kubectl exec -n rook-ceph $OSD_POD -c osd -- ss -tnp
```

Sample output:

```text
State    Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
ESTAB    0       0       10.0.0.1:43210     10.0.0.2:6800      pid=123,fd=15
ESTAB    0       0       10.0.0.1:43212     10.0.0.3:6800      pid=123,fd=16
ESTAB    0       0       10.0.0.1:43214     10.0.0.1:3300      pid=123,fd=17
```

## Measure RTT Between OSD Nodes

From the toolbox pod, test RTT to peer OSD pods:

```bash
OSD_IPS=$(kubectl get pods -n rook-ceph -l app=rook-ceph-osd \
  -o jsonpath='{.items[*].status.podIP}')

for ip in $OSD_IPS; do
  kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
    ping -c 5 -q $ip | grep rtt
done
```

## Check TCP Retransmits

High TCP retransmit counts indicate packet loss on the network:

```bash
kubectl exec -n rook-ceph \
  $(kubectl get pod -n rook-ceph -l ceph_daemon_type=osd,ceph_daemon_id=0 -o name) \
  -c osd -- cat /proc/net/snmp | grep -A1 Tcp
```

Or with `ss` for per-connection retransmit counts:

```bash
kubectl exec -n rook-ceph \
  $(kubectl get pod -n rook-ceph -l ceph_daemon_type=osd,ceph_daemon_id=0 -o name) \
  -c osd -- ss -tnip | grep -A1 "10.0.0"
```

Look for `retrans` in the output:

```text
ESTAB 0 0 10.0.0.1:43210 10.0.0.2:6800
  cubic wscale:7,7 rto:204 rtt:0.472/0.354 ato:40 mss:1448 pmtu:1500 rcvmss:1448 advmss:1448 cwnd:10 bytes_sent:1234 bytes_retrans:0 bytes_acked:1234 segs_out:128 segs_in:127 data_segs_out:64 data_segs_in:63 send 245.1Mbps lastsnd:4 lastrcv:4 lastack:4 pacing_rate 294.1Mbps delivery_rate 245.1Mbps delivered:64 busy:68ms unacked:0 retrans:0/0
```

## Monitor Network Throughput

Check bandwidth utilization on the OSD network interface:

```bash
kubectl exec -n rook-ceph \
  $(kubectl get pod -n rook-ceph -l ceph_daemon_type=osd,ceph_daemon_id=0 -o name) \
  -c osd -- cat /proc/net/dev | awk 'NR>2 {print $1, "RX:", $2/1024/1024, "MB TX:", $10/1024/1024, "MB"}'
```

## Set Network Tuning Parameters in Rook

For high-throughput clusters, tune TCP settings via the `rook-config-override` ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [global]
    ms_tcp_nodelay = true
    ms_tcp_rcvbuf = 0
    ms_initial_backoff = 0.2
    ms_max_backoff = 15
```

## Summary

TCP statistics for Ceph connections are inspected using `ss -tnip` inside OSD pods to view RTT, retransmit counts, and connection state. Target sub-millisecond RTT between OSDs for optimal replication latency. Use `ping` to baseline network latency, monitor retransmit counts for packet loss indicators, and apply TCP tuning parameters via the Rook `rook-config-override` ConfigMap when running high-throughput workloads.
