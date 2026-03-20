# How to Verify LACP Negotiation on a Linux Bond

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Bonding, LACP, 802.3ad, Verification, Networking, Troubleshooting

Description: Verify that LACP (802.3ad) is successfully negotiating with the switch on a Linux bond interface by reading /proc/net/bonding and checking aggregator IDs.

## Introduction

When configuring LACP bonding (mode 4), it is critical to verify that LACP PDUs are being exchanged with the switch and that all interfaces are in the aggregated (collecting/distributing) state. Misconfigured LACP results in all traffic using only one slave or complete connectivity loss.

## Prerequisites

- Bond interface configured in mode 4 (802.3ad)
- Switch LACP port-channel configured on the connected ports
- Root access

## Check LACP Status in /proc/net/bonding

```bash
cat /proc/net/bonding/bond0
```

A successful LACP negotiation shows:

```
Ethernet Channel Bonding Driver: v3.7.1

Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer3+4 (3)
MII Status: up
LACP rate: fast
Active Aggregator Info:
	Aggregator ID: 1
	Number of ports: 2
	Actor Key: 9
	Partner Key: 1
	Partner Mac Address: aa:bb:cc:dd:ee:ff

Slave Interface: eth0
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 00:11:22:33:44:55
LACP PDUs rx: 145
LACP PDUs tx: 145
Aggregator ID: 1
Actor Churn State: monitoring
Partner Churn State: monitoring
Actor Churned Count: 0
Partner Churned Count: 0

Slave Interface: eth1
MII Status: up
LACP PDUs rx: 143
LACP PDUs tx: 143
Aggregator ID: 1
```

## Key Indicators of Successful LACP

```bash
# 1. Both slaves have the same Aggregator ID
grep "Aggregator ID" /proc/net/bonding/bond0
# Should show the same ID for all slaves

# 2. LACP PDUs are being sent and received
grep "LACP PDUs" /proc/net/bonding/bond0
# Both rx and tx should be non-zero and increasing

# 3. "Number of ports" matches the number of slaves
grep "Number of ports" /proc/net/bonding/bond0
# Should show 2 (or however many slaves you have)
```

## Signs of LACP Negotiation Failure

```bash
# If aggregator IDs differ, interfaces are NOT in the same aggregation group
# Example of failure:
# Aggregator ID: 1  (eth0 - one group)
# Aggregator ID: 2  (eth1 - different group = NOT AGGREGATED)

# Check PDU counts — if tx is high but rx is 0, switch is not responding
grep "LACP PDUs" /proc/net/bonding/bond0
```

## Monitor LACP PDU Exchange

```bash
# Watch PDU counts increase over time (confirms active negotiation)
watch -n 1 "grep 'LACP PDUs' /proc/net/bonding/bond0"
```

## Capture LACP PDUs with tcpdump

```bash
# LACP PDUs use EtherType 0x8809 (slow protocols)
tcpdump -i eth0 ether proto 0x8809

# Filter specifically for LACP
tcpdump -i eth0 ether proto 0x8809 -e -v
```

## Common LACP Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| Different Aggregator IDs | Switch not in LACP mode | Configure switch port-channel with LACP |
| PDU rx = 0 | Switch not sending PDUs | Verify switch LACP configuration |
| Only 1 of 2 slaves up | Speed/duplex mismatch | Match speeds on all ports |
| LACP rate mismatch | Slow vs fast LACP rate | Match LACP rate on both sides |

## Conclusion

LACP negotiation is verified through `/proc/net/bonding/bond0`. Look for all slaves sharing the same Aggregator ID, increasing LACP PDU rx/tx counts, and the "Number of ports" matching your slave count. If any slave has a different Aggregator ID, that slave is not in the aggregation group and LACP negotiation with the switch has failed for that port.
