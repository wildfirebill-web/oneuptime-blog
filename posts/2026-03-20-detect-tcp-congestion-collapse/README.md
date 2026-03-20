# How to Detect TCP Congestion Collapse in Your Network

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: TCP, Congestion, Networking, Performance, Monitoring, Linux

Description: Identify TCP congestion collapse where network utilization drops dramatically due to excessive retransmissions, and apply congestion management to restore throughput.

## Introduction

TCP congestion collapse is a state where network utilization drops to near zero even though links are 100% utilized - because all the traffic is retransmissions of lost data rather than new data delivery. It was a major problem in early internet history and can still occur in heavily overloaded networks. Detection requires monitoring retransmission rates, goodput (useful throughput), and queue depths simultaneously.

## Signs of Congestion Collapse

```bash
# Sign 1: Interface utilization near 100% but application throughput very low

watch -n 1 "ip -s link show eth0 | grep 'TX bytes'"

# Compare interface byte rate vs application data rate:
# If interface shows 950 Mbps but applications achieve only 50 Mbps → collapse

# Sign 2: Very high TCP retransmission rate
nstat | grep TcpRetransSegs
# Normal: retransmit rate < 1% of segments
# Congestion collapse: 50-90% of segments are retransmissions!

# Sign 3: RTT increasing dramatically under load
ping -c 100 -i 0.1 10.20.0.5 | tail -3
# Normal: 1-5ms
# Under collapse: 500ms+ (queues building up)

# Sign 4: CWND oscillating rapidly
watch -n 0.5 "ss -tin state established | grep -oP 'snd_cwnd:\K\d+'"
# Should grow steadily; collapse shows rapid drops to 1
```

## Measuring Goodput vs. Throughput

```bash
# Throughput = total bytes sent including retransmissions
# Goodput = useful bytes actually delivered

# Use iperf3 to compare sender vs receiver rates
iperf3 -c 10.20.0.5 -t 30
# If sender rate >> receiver rate: retransmissions eating bandwidth

# Monitor retransmission ratio
BEFORE=$(nstat -z | awk '/TcpRetransSegs/{print $2}')
iperf3 -c 10.20.0.5 -t 30 &>/dev/null
AFTER=$(nstat -z | awk '/TcpRetransSegs/{print $2}')
echo "Retransmissions during test: $((AFTER - BEFORE))"
```

## Detecting Using Kernel Counters

```bash
#!/bin/bash
# Monitor for congestion collapse indicators

while true; do
    RETRANS=$(nstat -z 2>/dev/null | awk '/TcpRetransSegs/{r=$2} /TcpOutSegs/{o=$2}
      END{if(o>0) printf "%.1f%%", r/o*100; else print "0%"}')
    RX_DROPS=$(ip -s link show eth0 | awk '/RX:/{getline; print $4}')
    echo "$(date +%H:%M:%S) Retrans: $RETRANS  RX drops: $RX_DROPS"
    sleep 5
done
```

## Causes of Congestion Collapse

```bash
# Cause 1: Insufficient queue management (no AQM)
# Fix: enable Active Queue Management (AQM) on the bottleneck interface
tc qdisc replace dev eth0 root fq_codel   # FQ-CoDel reduces latency under load
# or
tc qdisc replace dev eth0 root cake bandwidth 1Gbit  # CAKE with rate limiting

# Cause 2: Single bottleneck link overwhelmed by too many flows
# Fix: use fair queueing
tc qdisc replace dev eth0 root fq

# Cause 3: Applications not respecting TCP backpressure
# Fix: add flow control in applications, use connection pooling
```

## Preventing Congestion Collapse

```bash
# Enable fair queueing with flow-fair pacing (BBR works best with this)
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr

# Enable ECN so routers can signal congestion before dropping
sysctl -w net.ipv4.tcp_ecn=1

# For edge devices, deploy fq_codel or cake on bottleneck interfaces
tc qdisc replace dev wan0 root fq_codel target 5ms interval 100ms
```

## Conclusion

TCP congestion collapse is diagnosed through simultaneous monitoring of retransmission rates and goodput. High interface utilization with low application throughput is the key signal. Prevention focuses on active queue management (fq_codel, cake), fair queueing, and using BBR congestion control which is more resistant to congestion collapse than loss-based algorithms.
