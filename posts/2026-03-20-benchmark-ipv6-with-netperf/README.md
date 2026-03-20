# How to Benchmark IPv6 with netperf

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, netperf, Benchmarking, Networking, Performance, TCP

Description: Use netperf to benchmark IPv6 network performance with TCP stream tests, TCP request/response tests, and UDP unidirectional throughput measurements.

## Introduction

netperf is a network benchmarking tool developed by HP that complements iperf3. It excels at measuring request/response performance and transaction throughput — patterns that better represent real application workloads than pure bulk throughput tests.

## Prerequisites

```bash
# Install netperf
sudo apt-get install netperf   # Debian/Ubuntu
sudo yum install netperf       # RHEL/CentOS

# Start the netserver on the remote host
netserver -6 -p 12865
```

## Step 1: TCP Stream Throughput (IPv6)

```bash
# Basic TCP bulk throughput test — IPv6
# -H specifies host with address family
netperf -6 -H 2001:db8::1 -t TCP_STREAM -l 30

# With explicit port and test duration
netperf -6 \
  -H 2001:db8::1,6 \  # ,6 = AF_INET6
  -p 12865 \
  -t TCP_STREAM \
  -l 30 \             # 30 seconds
  -- \
  -m 16384            # 16KB send message size

# Output:
# MIGRATED TCP STREAM TEST from 0.0.0.0 (0.0.0.0) to 2001:db8::1 () ...
# Recv   Send    Send
# Socket Socket  Message  Elapsed
# Size   Size    Size     Time    Throughput
# bytes  bytes   bytes    secs.   10^6bits/sec
# 87380  16384   16384    30.00    9412.34
```

## Step 2: TCP Request/Response (Transaction Rate)

This test measures how many short request-response cycles per second the network supports — crucial for microservices.

```bash
# TCP_RR — request/response test (measures transaction rate)
netperf -6 -H 2001:db8::1 -t TCP_RR -l 30 \
  -- \
  -r 64,64    # 64-byte request, 64-byte response

# Output shows transactions/second:
# Trans. Rate   Latency
# Trans/sec     usec/trans
# 85234.12      11.73

# Variable request sizes
netperf -6 -H 2001:db8::1 -t TCP_RR -l 30 \
  -- -r 512,512    # Simulate a typical API call size
```

## Step 3: TCP Connect/Request/Response (Connection Overhead)

```bash
# TCP_CRR — each transaction uses a new connection
# Measures the overhead of connection setup + request
netperf -6 -H 2001:db8::1 -t TCP_CRR -l 30 \
  -- -r 64,64

# Compare TCP_RR vs TCP_CRR to quantify connection setup cost:
echo "=== TCP_RR (persistent connection) ==="
netperf -6 -H 2001:db8::1 -t TCP_RR -l 15 -- -r 64,64

echo "=== TCP_CRR (new connection per transaction) ==="
netperf -6 -H 2001:db8::1 -t TCP_CRR -l 15 -- -r 64,64
```

## Step 4: UDP Unidirectional Throughput

```bash
# UDP_STREAM — measure maximum UDP throughput
netperf -6 -H 2001:db8::1 -t UDP_STREAM -l 30 \
  -- -m 1400    # 1400-byte datagrams

# UDP_RR — UDP request/response (measures round-trip)
netperf -6 -H 2001:db8::1 -t UDP_RR -l 30 \
  -- -r 160,160  # VoIP-sized packets
```

## Step 5: Automated Benchmark Script

```bash
#!/bin/bash
# ipv6-netperf-suite.sh

SERVER="2001:db8::1"
DURATION=30

echo "=== IPv6 netperf Benchmark Suite ==="
printf "%-30s %15s %15s\n" "Test" "Throughput" "Latency"
printf "%-30s %15s %15s\n" "----" "----------" "-------"

# TCP Stream
RESULT=$(netperf -6 -H "$SERVER" -t TCP_STREAM -l $DURATION 2>/dev/null | tail -1)
THROUGHPUT=$(echo "$RESULT" | awk '{print $NF}')
printf "%-30s %12s Mbps %15s\n" "TCP Stream" "$THROUGHPUT" "N/A"

# TCP RR
RESULT=$(netperf -6 -H "$SERVER" -t TCP_RR -l $DURATION -- -r 64,64 2>/dev/null | tail -1)
TPS=$(echo "$RESULT" | awk '{print $1}')
LAT=$(echo "$RESULT" | awk '{print $2}')
printf "%-30s %11s TPS %12s usec\n" "TCP RR (64B)" "$TPS" "$LAT"

# TCP CRR
RESULT=$(netperf -6 -H "$SERVER" -t TCP_CRR -l $DURATION -- -r 64,64 2>/dev/null | tail -1)
TPS=$(echo "$RESULT" | awk '{print $1}')
LAT=$(echo "$RESULT" | awk '{print $2}')
printf "%-30s %11s TPS %12s usec\n" "TCP CRR (64B)" "$TPS" "$LAT"

# UDP Stream
RESULT=$(netperf -6 -H "$SERVER" -t UDP_STREAM -l $DURATION -- -m 1400 2>/dev/null | tail -1)
THROUGHPUT=$(echo "$RESULT" | awk '{print $NF}')
printf "%-30s %12s Mbps %15s\n" "UDP Stream (1400B)" "$THROUGHPUT" "N/A"
```

## Conclusion

netperf's request/response tests provide insight into IPv6 latency characteristics that pure throughput tests miss. TCP_RR transaction rates directly correlate with microservice call performance. Use these baselines alongside OneUptime's synthetic monitoring to establish and maintain IPv6 performance SLOs.
