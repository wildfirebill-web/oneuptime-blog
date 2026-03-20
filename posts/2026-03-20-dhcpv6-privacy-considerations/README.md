# How to Understand DHCPv6 Privacy Considerations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCPv6, IPv6, Privacy, DUID, RFC 7844

Description: Understand the privacy implications of DHCPv6, including DUID tracking, address stability issues, and how RFC 7844 anonymous stateless profiles help protect user identity.

## Overview

DHCPv6 introduces privacy concerns that do not exist in the same form with DHCPv4 or SLAAC. Clients expose persistent identifiers (DUIDs) in every DHCPv6 message, enabling cross-network tracking. RFC 7844 defines anonymity profiles to mitigate this.

## Why DHCPv6 Has Privacy Issues

When a DHCPv6 client sends a Solicit or Request, it includes a **DUID** (DHCP Unique Identifier) in the Client Identifier option. DUIDs are often based on the MAC address or a UUID tied to the hardware.

```
# Typical DUID-LL (type 3) based on MAC address
Client Identifier: 00:03:00:01:aa:bb:cc:dd:ee:ff
                   ^^^^^ ^^^^^ ^^^^^^^^^^^^^^^^
                   type  hw   MAC address
```

This DUID is the same on every network the client visits, allowing any network operator to:
- Track a device across different locations
- Correlate visits to the same hotspot over time
- Link DHCPv6 requests to a specific device or user

## RFC 7844: Anonymity Profiles for DHCP

RFC 7844 defines "anonymity profiles" that minimize the information revealed in DHCP messages.

Key recommendations for DHCPv6:
1. Use a randomly generated DUID (DUID-UUID with random UUID) per network
2. Do not include the Client FQDN (hostname) option
3. Do not include the User Class option
4. Use a Rapid Commit if supported to reduce message exposure
5. Request only necessary options

## Configuring an Anonymous DUID on Linux

By default, `dhclient` stores a persistent DUID. You can override it with a random one per connection:

```bash
# Generate a random DUID-UUID for privacy
python3 -c "
import uuid, struct
u = uuid.uuid4()
# DUID type 4 (UUID) = 0x0004
duid = b'\x00\x04' + u.bytes
print(':'.join(f'{b:02x}' for b in duid))
"

# Set a random DUID in dhclient config
# /etc/dhcp/dhclient.conf
send dhcp6.client-id 00:04:<random-uuid-bytes>;
```

## NetworkManager and Privacy

NetworkManager supports RFC 7844 anonymity profiles:

```bash
# Enable anonymity profile for a connection via nmcli
nmcli connection modify "MyWifi" ipv6.dhcp-iaid "stable-privacy"

# Or use temporary random DUID
nmcli connection modify "MyWifi" ipv6.dhcp-duid "ll"  # link-local based

# Alternatively, edit the connection file
# /etc/NetworkManager/system-connections/MyWifi.nmconnection
[ipv6]
dhcp-duid=stable-privacy
dhcp-iaid=stable-privacy
```

## IPv6 Address Stability vs. Privacy

Even with DHCPv6, the assigned address may be stable and trackable. RFC 8064 and RFC 7217 address this for SLAAC, but for DHCPv6:

- A persistent lease means the same address is renewed each time
- This enables long-term tracking even without the DUID

To mitigate: configure shorter lease times or use privacy addresses on the operating system level alongside DHCPv6.

## Information Exposed in DHCPv6 Messages

| DHCPv6 Option | Privacy Risk | Recommendation |
|---------------|-------------|----------------|
| Client Identifier (DUID) | Persistent device fingerprint | Use random DUID per network |
| Client FQDN (option 39) | Reveals hostname | Suppress in privacy mode |
| User Class (option 15) | Reveals OS/vendor | Suppress in privacy mode |
| Vendor Class (option 16) | Reveals device type | Suppress in privacy mode |
| Requested Options | Fingerprints implementation | Minimize to essential options |

## ISP Logging Considerations

ISPs log DHCPv6 assignments including DUIDs. This creates a permanent record linking a DUID to an IPv6 prefix. From a compliance standpoint (GDPR, etc.), ISPs must treat DUIDs as personal data.

## Summary

DHCPv6 privacy risks stem primarily from the persistent DUID included in every message. RFC 7844 defines anonymity profiles that randomize identifiers and minimize option exposure. Users and administrators should configure random DUIDs and suppress hostname options in environments where tracking is a concern.
