# How to Monitor Traffic Inside a Network Namespace with tcpdump

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Namespaces, tcpdump, Traffic Analysis, Networking, Monitoring, Packet Capture

Description: Use tcpdump inside a network namespace to capture and analyze traffic on namespace-specific interfaces, isolated from host network traffic.

## Introduction

tcpdump can be run inside a network namespace using `ip netns exec`, capturing only the traffic on interfaces within that namespace. This is extremely useful for debugging namespace connectivity, monitoring isolated services, and verifying that traffic is flowing correctly through veth pairs.

## Prerequisites

- tcpdump installed (`apt install tcpdump` or `yum install tcpdump`)
- A configured network namespace with interfaces
- Root access

## Basic Packet Capture Inside a Namespace

```bash
# Capture all traffic on all interfaces inside ns1

ip netns exec ns1 tcpdump -i any

# Capture traffic on a specific interface inside the namespace
ip netns exec ns1 tcpdump -i veth-ns

# Capture with no DNS resolution (-n) and verbose output (-v)
ip netns exec ns1 tcpdump -i any -n -v
```

## Filter Specific Traffic

```bash
# Capture only ICMP (ping) traffic inside the namespace
ip netns exec ns1 tcpdump -i any icmp

# Capture TCP traffic on port 80
ip netns exec ns1 tcpdump -i any tcp port 80

# Capture traffic to/from a specific host
ip netns exec ns1 tcpdump -i any host 10.0.0.1

# Capture DNS traffic
ip netns exec ns1 tcpdump -i any udp port 53
```

## Save Captures to a File

```bash
# Write capture to a pcap file for analysis in Wireshark
ip netns exec ns1 tcpdump -i any -w /tmp/ns1-capture.pcap

# Capture 100 packets and stop
ip netns exec ns1 tcpdump -i any -c 100 -w /tmp/ns1-100pkts.pcap

# Read the saved capture
tcpdump -r /tmp/ns1-capture.pcap
```

## Monitor the veth Pair from Both Sides

You can run tcpdump on both sides of a veth pair simultaneously to trace packet flow:

```bash
# Terminal 1: Capture on the host side of the veth pair
tcpdump -i veth-host -n

# Terminal 2: Capture on the namespace side
ip netns exec ns1 tcpdump -i veth-ns -n

# Terminal 3: Generate traffic from inside the namespace
ip netns exec ns1 ping -c 5 10.0.0.1
```

## Capture and Display Packet Content

```bash
# Show ASCII content of packets (useful for HTTP debugging)
ip netns exec ns1 tcpdump -i any -A tcp port 80

# Show hex and ASCII
ip netns exec ns1 tcpdump -i any -X tcp port 80
```

## Real-Time Traffic Summary

```bash
# Show packet statistics with timestamps
ip netns exec ns1 tcpdump -i any -tttt -q

# Show only source/destination without payload
ip netns exec ns1 tcpdump -i any -q
```

## Debugging Namespace Connectivity Issues

A common workflow to verify traffic flow:

```bash
# Step 1: Start a capture inside the namespace
ip netns exec ns1 tcpdump -i any icmp &

# Step 2: Trigger traffic
ip netns exec ns1 ping -c 3 10.0.0.1

# Step 3: If no packets appear in the capture, the issue is
# that the packets are not reaching the namespace interface

# Step 4: Check the host-side veth for comparison
tcpdump -i veth-host icmp
```

## Capture with Context Using nsenter

For container namespaces (unnamed), use nsenter:

```bash
# Capture inside a container's network namespace by PID
CONTAINER_PID=$(docker inspect --format '{{.State.Pid}}' mycontainer)
nsenter --target $CONTAINER_PID --net -- tcpdump -i eth0 -n
```

## Conclusion

Running tcpdump with `ip netns exec` provides full packet capture capability inside any network namespace. Capture on specific interfaces to isolate traffic, save to pcap files for deeper analysis, and use dual-capture (both sides of a veth pair) to trace exactly where packets are flowing or being dropped. This is an indispensable tool for debugging namespace networking issues.
