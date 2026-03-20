# How to Configure NetFlow Export on a Juniper Router

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NetFlow, Juniper, Junos, Traffic Analysis, Flow Monitoring

Description: Learn how to configure active flow monitoring and NetFlow/IPFIX export on Juniper routers running Junos OS for network traffic visibility.

## Juniper Flow Monitoring Architecture

Juniper Junos uses a different configuration model from Cisco IOS. Flow monitoring is configured under `forwarding-options` as a "sampling" or "flow-server" configuration. Juniper supports sampling-based flow export (similar to sFlow sampling) and inline monitoring.

## Step 1: Configure a Flow Server (Collector Destination)

Define the NetFlow collector:

```
# Junos configuration hierarchy
set forwarding-options flow-monitoring version9 template IPV4_TEMPLATE ip-headers
set forwarding-options flow-monitoring version9 template IPV4_TEMPLATE transport-ports
set forwarding-options flow-monitoring version9 template IPV4_TEMPLATE protocol
set forwarding-options flow-monitoring version9 template IPV4_TEMPLATE counter

# Define the flow export destination
set forwarding-options flow-monitoring version9 flow-export-destination NETFLOW_COLLECTOR
set forwarding-options flow-monitoring version9 flow-export-destination NETFLOW_COLLECTOR version version9
set forwarding-options flow-monitoring version9 flow-export-destination NETFLOW_COLLECTOR template IPV4_TEMPLATE
set forwarding-options flow-monitoring version9 flow-export-destination NETFLOW_COLLECTOR remote-address 192.168.1.200
set forwarding-options flow-monitoring version9 flow-export-destination NETFLOW_COLLECTOR remote-port 2055
set forwarding-options flow-monitoring version9 flow-export-destination NETFLOW_COLLECTOR source-address 10.0.0.1
```

## Step 2: Configure Sampling

Define the sampling policy—how many packets to sample:

```
# Configure 1-in-1000 packet sampling
set forwarding-options sampling instance NETFLOW_SAMPLE input rate 1000
set forwarding-options sampling instance NETFLOW_SAMPLE input run-length 0
set forwarding-options sampling instance NETFLOW_SAMPLE family inet output flow-server NETFLOW_COLLECTOR
set forwarding-options sampling instance NETFLOW_SAMPLE family inet output inline-jflow source-address 10.0.0.1
```

## Step 3: Apply Sampling to Interfaces

```
# Apply to the WAN interface (ingress and egress)
set interfaces ge-0/0/0 unit 0 family inet sampling input
set interfaces ge-0/0/0 unit 0 family inet sampling output

# Apply to LAN interface
set interfaces ge-0/0/1 unit 0 family inet sampling input
```

## Step 4: Alternative - Configure Using Firewall Filter (for Full Flow Export)

For complete NetFlow (not sampled), use a firewall filter with syslog action:

```
# Create a filter that matches all traffic and samples it
set firewall family inet filter NETFLOW_EXPORT term all-traffic from protocol [ tcp udp icmp ]
set firewall family inet filter NETFLOW_EXPORT term all-traffic then sample
set firewall family inet filter NETFLOW_EXPORT term all-traffic then accept

set firewall family inet filter NETFLOW_EXPORT term default then accept

# Apply to interface
set interfaces ge-0/0/0 unit 0 family inet filter input NETFLOW_EXPORT
set interfaces ge-0/0/0 unit 0 family inet filter output NETFLOW_EXPORT
```

## Step 5: Configure IPFIX Export (v9 Compatible)

To export using IPFIX format instead of NetFlow v9:

```
# Use IPFIX protocol
set forwarding-options flow-monitoring version-ipfix template IPV4_IPFIX ip-headers
set forwarding-options flow-monitoring version-ipfix template IPV4_IPFIX transport-ports
set forwarding-options flow-monitoring version-ipfix flow-export-destination IPFIX_COLLECTOR
set forwarding-options flow-monitoring version-ipfix flow-export-destination IPFIX_COLLECTOR version ipfix
set forwarding-options flow-monitoring version-ipfix flow-export-destination IPFIX_COLLECTOR template IPV4_IPFIX
set forwarding-options flow-monitoring version-ipfix flow-export-destination IPFIX_COLLECTOR remote-address 192.168.1.200
set forwarding-options flow-monitoring version-ipfix flow-export-destination IPFIX_COLLECTOR remote-port 4739
```

## Step 6: Verify Flow Export

```
# Show active sampling configuration
show class-of-service interface ge-0/0/0

# Show flow monitoring statistics
show services flow-monitoring statistics

# Check for exported flows
show services flow-monitoring flow-table

# Verify on the collector
sudo tcpdump -i any udp port 2055 -n -c 5
```

## Step 7: View Configuration as Full Hierarchy

```
# Show complete flow monitoring configuration
show configuration forwarding-options sampling
show configuration forwarding-options flow-monitoring
```

## Juniper vs Cisco NetFlow Configuration Comparison

| Aspect | Cisco IOS | Juniper Junos |
|---|---|---|
| Enable on interface | `ip flow ingress` | `sampling input` |
| Export destination | `ip flow-export destination X` | Under `flow-monitoring` |
| Version | `ip flow-export version 9` | Template-based |
| Sampling rate | `ip flow-cache` | `sampling rate 1000` |

## Conclusion

NetFlow export on Juniper Junos uses the `forwarding-options flow-monitoring` and `forwarding-options sampling` hierarchy. Define the flow template fields, the collector destination, and the sampling rate, then apply sampling to interfaces using `family inet sampling input/output`. Verify export with `show services flow-monitoring statistics` and confirm the collector is receiving data.
