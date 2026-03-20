# How to Simulate Packet Reordering on IPv4 with tc netem

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: tc, Netem, IPv4, Packet Reordering, Network Testing, Linux

Description: Use tc netem to deliberately reorder IPv4 packets on a Linux interface to test how TCP stacks and applications handle out-of-order delivery.

TCP handles packet reordering transparently, but high reorder rates can cause unnecessary retransmissions and performance degradation. Testing reordering helps verify TCP implementation behavior and application resilience.

## How netem Reordering Works

netem implements reordering by delaying most packets by a fixed delay but sending a fraction of them immediately (before the delayed ones), causing them to arrive in a different order.

## Basic Packet Reordering

```bash
# Reorder 25% of packets, with a 10ms base delay for the others

# This means 25% of packets are sent immediately, 75% are delayed 10ms
sudo tc qdisc add dev eth0 root netem \
  delay 10ms \
  reorder 25% 50%

# The second number (50%) is the correlation to previous packet's delay
```

## Understanding Reorder Parameters

```text
netem reorder <percent> [correlation]
- percent: probability a packet is sent immediately (before delayed packets)
- correlation: how much the delay of consecutive packets are related
```

## Simulating Out-of-Order Delivery

```bash
# More aggressive reordering: 50% of packets sent immediately
sudo tc qdisc add dev eth0 root netem delay 100ms reorder 50%

# With correlation: consecutive packets tend to be either all delayed or all immediate
sudo tc qdisc add dev eth0 root netem delay 100ms reorder 50% 75%
```

## Gap-Based Reordering

```bash
# Reorder every Nth packet (gap-based), 100ms delay, every 5th packet sent immediately
sudo tc qdisc add dev eth0 root netem gap 5 delay 100ms
```

## Testing TCP Behavior Under Reordering

```bash
# Apply reordering
sudo tc qdisc add dev eth0 root netem delay 50ms reorder 30%

# Generate TCP traffic to observe behavior
iperf3 -c <SERVER_IP> -t 30 -P 4

# Check TCP retransmission statistics on the receiver
# On receiver:
ss -s | grep -i retransmit
netstat -s | grep -i "segments retransmit"
```

## Checking TCP Retransmission Rate with tcpdump

```bash
# Capture and count TCP retransmissions
sudo tcpdump -i eth0 -n 'tcp[tcpflags] & (tcp-syn|tcp-fin) = 0 and tcp[4:4] = 0' \
  -c 1000 2>/dev/null | wc -l
```

## Combining with Loss and Delay

```bash
# Complex impairment: delay + reorder + loss
sudo tc qdisc add dev eth0 root netem \
  delay 80ms 20ms \
  loss 2% \
  reorder 15% 50%
```

## Observing Reordering Effects

```bash
# Use ss to check TCP internal state
ss -tin dst <SERVER_IP>
# Look for:
# sacked: number of SACK blocks (indicates received but out-of-order packets)
# retrans: retransmission count

# Monitor in real time
watch -n 1 "ss -tin dst <SERVER_IP> | grep -A2 ESTABLISHED"
```

## Cleanup

```bash
sudo tc qdisc del dev eth0 root
```

Packet reordering testing is particularly relevant for evaluating applications that use multiple concurrent TCP connections or UDP with custom reliability mechanisms, where out-of-order delivery handling must be explicitly verified.
