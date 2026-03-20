# How to Simulate Packet Corruption on IPv4 with tc netem

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: tc, netem, Packet Corruption, Network Testing, IPv4, Linux, QoS

Description: Use Linux tc netem to inject artificial packet corruption on a network interface to test application resilience and error handling under degraded IPv4 network conditions.

## Introduction

The `netem` (Network Emulator) qdisc in Linux's traffic control subsystem can simulate various types of network degradation: delay, packet loss, corruption, duplication, and reordering. Injecting packet corruption lets you verify that your application and its underlying protocols handle bit errors gracefully.

## Understanding Packet Corruption

Packet corruption flips random bits within a packet's payload. The receiving host's checksum validation will detect the corruption, causing the packet to be dropped and retransmitted at the TCP layer. This simulates poor physical links (noisy Ethernet, bad fiber, bit errors on wireless).

## Enabling Packet Corruption

Apply a netem qdisc to a network interface:

```bash
# Add netem qdisc with 0.1% packet corruption to eth0
sudo tc qdisc add dev eth0 root netem corrupt 0.1%

# Verify the rule was added
sudo tc qdisc show dev eth0
```

`corrupt 0.1%` means each packet has a 0.1% chance of having a single bit flipped.

## Increasing Corruption for Stress Testing

```bash
# Simulate a very poor link with 5% corruption
sudo tc qdisc add dev eth0 root netem corrupt 5%
```

## Combining Corruption with Other Network Impairments

You can simulate multiple conditions simultaneously:

```bash
# Combine corruption with packet loss and delay
sudo tc qdisc add dev eth0 root netem \
  delay 50ms 10ms \       # 50ms delay with ±10ms jitter
  loss 1% \               # 1% random packet loss
  corrupt 0.5% \          # 0.5% packet corruption
  duplicate 0.1%          # 0.1% packet duplication
```

## Modifying an Existing Rule

Use `change` instead of `add` to update an existing netem qdisc:

```bash
# Change corruption rate without removing the existing rule
sudo tc qdisc change dev eth0 root netem corrupt 2%
```

## Removing the Rule

```bash
# Remove all tc rules from the interface
sudo tc qdisc del dev eth0 root
```

## Verifying Corruption Effects

Monitor the impact with `netstat` or `ss`:

```bash
# Watch TCP retransmissions (evidence of corruption)
watch -n 1 'netstat -s | grep retransmit'

# Or use ss for per-socket retransmit counters
ss -ti dst 10.0.0.1

# Monitor with ping (look for increased error rates)
ping -f -c 10000 10.0.0.1 | tail -3
```

## Targeting Specific Traffic

Combine netem with a parent `prio` qdisc and filters to only corrupt specific flows:

```bash
# Create a priority qdisc with three bands
sudo tc qdisc add dev eth0 root handle 1: prio

# Add netem to band 3 (low-priority band)
sudo tc qdisc add dev eth0 parent 1:3 handle 30: netem corrupt 2%

# Classify traffic to the specific backend server to band 3
sudo tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
  match ip dst 10.0.1.50/32 \
  flowid 1:3
```

## Use Cases

- **Testing TCP resilience**: Verify that your application handles retransmissions and eventual delivery correctly
- **Validating protocol checksums**: Ensure checksums properly reject corrupted packets
- **CI/CD chaos testing**: Automate degraded network tests as part of your pipeline

## Conclusion

`tc netem corrupt` is a powerful tool for chaos engineering and resilience testing. Always apply it on non-production interfaces or in a dedicated test environment, and remember to remove the rule when testing is complete.
