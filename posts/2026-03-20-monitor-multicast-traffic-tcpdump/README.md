# How to Monitor Multicast Traffic with tcpdump

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Multicast, tcpdump, Linux, Network Monitoring, Packet Capture

Description: Use tcpdump to capture and analyze multicast traffic on a Linux system, filtering by multicast address ranges and decoding IGMP control messages.

## Introduction

Multicast traffic enables one-to-many communication, making it essential for streaming, routing protocols, and service discovery. Monitoring multicast with `tcpdump` lets you verify group membership, diagnose delivery failures, and audit traffic scope.

## Prerequisites

- Linux system with `tcpdump` installed (`sudo apt install tcpdump` or `sudo yum install tcpdump`)
- Root or `CAP_NET_RAW` capability
- A network interface receiving multicast traffic

## Capturing All Multicast Traffic

IPv4 multicast addresses fall in the `224.0.0.0/4` range. Use a BPF filter to isolate them:

```bash
# Capture all IPv4 multicast packets on eth0, show verbose output
sudo tcpdump -i eth0 -n -v "dst net 224.0.0.0/4"
```

The `-n` flag prevents DNS lookups, keeping output fast. `-v` adds decoded protocol fields.

## Filtering by Specific Group Address

When troubleshooting a known group:

```bash
# Capture traffic for a specific multicast group (e.g., 239.1.2.3)
sudo tcpdump -i eth0 -n "host 239.1.2.3"
```

## Capturing IGMP Control Messages

IGMP manages group membership. Protocol number 2 matches all IGMP packets:

```bash
# Capture IGMP messages only (protocol 2)
sudo tcpdump -i eth0 -n -v "ip proto 2"
```

Typical output shows IGMP Membership Reports (Join) and Leave messages:

```
12:01:05.123456 IP 192.168.1.50 > 224.0.0.22: igmp v3 report, 1 group record(s)
12:01:05.567890 IP 192.168.1.50 > 224.0.0.2: igmp leave 239.1.2.3
```

## Capturing Link-Local Multicast (224.0.0.0/24)

Link-local multicast is used by routing protocols (OSPF, RIP) and service discovery:

```bash
# Capture link-local multicast — these packets should NOT cross routers
sudo tcpdump -i eth0 -n "dst net 224.0.0.0/24"
```

## Writing Captures to a File for Analysis

For deeper analysis in Wireshark, save to a pcap file:

```bash
# Capture 1000 multicast packets and save to file
sudo tcpdump -i eth0 -n -c 1000 -w /tmp/multicast.pcap "dst net 224.0.0.0/4"

# Read it back with full verbosity
sudo tcpdump -r /tmp/multicast.pcap -n -vv
```

## Monitoring Multicast on All Interfaces

Use `any` as the interface to see multicast across all adapters (note: no BPF hardware offload on `any`):

```bash
sudo tcpdump -i any -n "dst net 224.0.0.0/4"
```

## Checking Multicast Traffic Rate

Pipe output to count packets per second:

```bash
# Count multicast packets arriving in 10 seconds
sudo tcpdump -i eth0 -n -q "dst net 224.0.0.0/4" 2>&1 | \
  awk 'BEGIN{t=systime()} /IP/{n++} END{print n" packets in "systime()-t"s"}'
```

## Common Multicast Addresses to Know

| Address | Use |
|---|---|
| 224.0.0.1 | All hosts on segment |
| 224.0.0.2 | All routers |
| 224.0.0.5 | OSPF routers |
| 224.0.0.251 | mDNS |
| 239.0.0.0/8 | Administratively scoped (private) |

## Conclusion

`tcpdump` with BPF multicast filters is a quick and powerful way to verify multicast delivery, inspect IGMP membership events, and capture traffic for offline analysis. Combine it with `-vv` for full packet decoding when diagnosing protocol-level issues.
