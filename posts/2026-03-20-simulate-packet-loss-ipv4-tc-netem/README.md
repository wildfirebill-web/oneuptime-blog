# How to Simulate Packet Loss on an IPv4 Interface Using tc netem

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: tc, Netem, IPv4, Packet Loss, Network Testing, Linux

Description: Use tc netem to simulate IPv4 packet loss with configurable rates and correlation to test application resilience under poor network conditions.

Simulating packet loss helps validate how TCP applications handle retransmissions, UDP applications handle missing data, and how quickly applications detect and recover from network degradation.

## Basic Packet Loss

```bash
# Drop 5% of all outgoing packets on eth0

sudo tc qdisc add dev eth0 root netem loss 5%

# Test with ping - some packets should not receive replies
ping -c 20 8.8.8.8

# Expected: ~1 out of 20 packets dropped (on the outgoing side)
# Actual packet loss ≈ 10% since server replies may also experience loss
```

## Correlated Packet Loss (Bursty Loss)

Real-world packet loss often comes in bursts, not randomly. Correlation links the probability of a drop to whether the previous packet was dropped:

```bash
# 10% loss with 25% correlation (bursty pattern)
sudo tc qdisc add dev eth0 root netem loss 10% 25%
```

## Gilbert-Elliott Loss Model (Realistic Bursty Loss)

```bash
# Two-state Markov model for realistic bursty packet loss
# p: probability of entering loss state, r: probability of leaving loss state
# 1-h: loss probability in good state, 1-k: loss probability in loss state
sudo tc qdisc add dev eth0 root netem loss gemodel p 10% r 80% 1-h 0% 1-k 100%
```

## Combined Loss with Delay

```bash
# Simulate a bad cellular connection: 200ms latency + 3% packet loss
sudo tc qdisc add dev eth0 root netem delay 200ms 50ms loss 3%
```

## Applying Loss on the Loopback for Local Testing

```bash
# Useful for testing local services without needing a remote endpoint
sudo tc qdisc add dev lo root netem loss 2%

# Test a local server
curl http://localhost:8080

# Remove when done
sudo tc qdisc del dev lo root
```

## Monitoring Loss Effects

```bash
# Use ping with count and flood option to measure loss
ping -c 100 -f 8.8.8.8

# Use mtr for continuous monitoring with per-hop loss stats
mtr --report --report-cycles 50 8.8.8.8

# Capture with tcpdump and analyze TCP retransmissions
sudo tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-syn != 0' -c 50
```

## Testing TCP Retransmission Behavior

```bash
# Apply 5% loss and then do a large file transfer
sudo tc qdisc add dev eth0 root netem loss 5%

# Transfer a large file and observe retransmissions
scp largefile.bin user@remote:/tmp/

# Check TCP retransmission counters
netstat -s | grep retransmit
# or
ss -s
```

## Removing Loss Simulation

```bash
# Remove all netem rules
sudo tc qdisc del dev eth0 root

# Or change to zero loss (effectively removes loss but keeps netem)
sudo tc qdisc change dev eth0 root netem loss 0%
```

Packet loss simulation is particularly valuable for testing UDP-based applications (video streaming, gaming, VoIP) and verifying that TCP connection recovery works correctly in your application stack.
