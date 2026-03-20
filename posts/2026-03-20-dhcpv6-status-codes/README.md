# How to Understand DHCPv6 Status Codes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, IPv6, Troubleshooting, Status Codes, RFC 8415

Description: A reference guide to DHCPv6 status codes, what they mean, and how to identify and resolve them during address assignment failures.

## Overview

DHCPv6 status codes are included in Reply messages to inform the client of the outcome of its request. Understanding these codes is essential for diagnosing why a client failed to receive an address or prefix.

## DHCPv6 Status Code Table

| Code | Name | Description |
|------|------|-------------|
| 0 | Success | The request was processed successfully |
| 1 | UnspecFail | An unspecified failure occurred on the server |
| 2 | NoAddrsAvail | No addresses are available in the requested pool |
| 3 | NoBinding | The requested binding (lease) does not exist on the server |
| 4 | NotOnLink | The requested prefix is not on the client's link |
| 5 | UseMulticast | Client must use multicast to communicate with the server |
| 6 | NoPrefixAvail | No prefixes are available for prefix delegation |
| 7–65535 | Reserved / Vendor | Defined by IANA or vendor extensions |

## Status Code Location in Messages

Status codes appear inside the **IA_NA**, **IA_TA**, or **IA_PD** options within a Reply message — not at the top level of the DHCPv6 message. This means you must look inside the IA option to find the status.

## Diagnosing Status Codes with tcpdump

```bash
# Capture DHCPv6 Reply messages and show full option decode
sudo tcpdump -i eth0 -n -vvv "udp port 547 and ip6[40] == 7"

# ip6[40] == 7 filters for DHCPv6 Reply messages (message type 7)
# Look in the output for "status-code" option
```

## Diagnosing with tshark

```bash
# Extract status codes from all DHCPv6 Reply messages
tshark -r /tmp/dhcpv6.pcap -Y "dhcpv6.msgtype == 7" \
  -T fields \
  -e dhcpv6.status_code \
  -e dhcpv6.status_message
```

## Common Status Codes and Resolutions

### NoAddrsAvail (Code 2)

The server has no available addresses in the pool.

```bash
# Check Kea pool utilization
curl -s -X POST http://localhost:8000/ \
  -d '{"command": "subnet6-get",
       "arguments": {"id": 1},
       "service": ["dhcp6"]}' | jq .

# Expand the pool range in kea-dhcp6.conf
# "pools": [{ "pool": "2001:db8::100 - 2001:db8::fff" }]
```

### NoBinding (Code 3)

The client sent a Renew or Rebind for a lease the server doesn't know about.

```bash
# This often happens after a server wipe or migration
# Resolution: client should send a Solicit to get a new address
sudo dhclient -6 -r eth0 && sudo dhclient -6 eth0
```

### NotOnLink (Code 4)

The server is telling the client that the prefix in the Confirm message is not valid for this link.

```bash
# This typically means the client moved to a new network
# Resolution: the client must start fresh with a Solicit
# Check the client's current IA_NA in the Reply to see the correct prefix
```

### UseMulticast (Code 5)

The client sent a message via unicast when it should have used multicast.

```bash
# Clients should send Solicit/Request/Confirm to ff02::1:2
# This code tells the client to retry using multicast
# Check client DHCPv6 implementation for unicast enforcement bugs
```

### NoPrefixAvail (Code 6)

No prefixes are available for delegation.

```bash
# Check the PD pool in Kea
curl -s -X POST http://localhost:8000/ \
  -d '{"command": "statistic-get-all", "service": ["dhcp6"]}' | \
  jq '.[0].arguments | to_entries[] | select(.key | contains("pd"))'
```

## Forcing a Status Code in Testing

In a test lab using Kea, you can simulate pool exhaustion by setting a small pool and verifying the client receives NoAddrsAvail:

```json
// Tiny pool for testing exhaustion
"pools": [{ "pool": "2001:db8::1 - 2001:db8::1" }]
```

Once the single address is leased, the next client will receive a Reply with status code 2 (NoAddrsAvail).

## Summary

DHCPv6 status codes embedded in IA options clearly indicate why address assignment succeeded or failed. The most common issues are exhausted pools (NoAddrsAvail), stale leases (NoBinding), and network changes (NotOnLink). Always inspect status codes when clients fail to get IPv6 addresses.
