# How to Configure IPv6 for CMTS (Cable Modem Termination Systems)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, CMTS, DOCSIS, Cable, ISP, DHCPv6

Description: Configure IPv6 on Cable Modem Termination Systems (CMTS) using DOCSIS 3.0/3.1 with DHCPv6 prefix delegation for cable internet subscribers.

## CMTS and IPv6

Cable networks use DOCSIS (Data Over Cable Service Interface Specification). DOCSIS 3.0 and 3.1 support IPv6 natively. The CMTS is the ISP-side device that terminates cable modem connections and manages subscriber IPv6 addressing.

## DOCSIS IPv6 Architecture

```mermaid
flowchart LR
    CM[Cable Modem] -- DOCSIS --> CMTS
    CMTS -- DHCPv6 Relay --> DHCP[DHCPv6 Server]
    CMTS -- BGP --> Core[ISP Core Network]
    CM --> CPE[Customer Router\n(gets /56 via PD)]
```

## Cisco CMTS (uBR10012) IPv6 Configuration

Configure IPv6 on the Cable interface and enable DHCPv6 relay:

```text
! Enable IPv6 on cable interface
interface Cable1/0/0
 ipv6 address 2001:db8:cmts:1::1/64
 ipv6 nd ra-interval 30
 ! Enable DHCPv6 relay for prefix delegation
 ipv6 dhcp relay destination 2001:db8:dhcp::10

! Configure IPv6 routing
ipv6 unicast-routing
ipv6 cef

! Loopback for CMTS management
interface Loopback0
 ipv6 address 2001:db8:mgmt::1/128
```

## DOCSIS Configuration File for IPv6 Cable Modems

DOCSIS configuration files sent to cable modems must include IPv6 parameters:

```text
# DOCSIS config file (binary, shown in text format)

# Enable IPv6 on the cable modem
NetworkAccess 1;

# IPv6 CPE assignment via DHCPv6
IPv6CPEEnabled 1;

# Allow prefix delegation to CPE
IPv6PrefixDelegationEnabled 1;
```

## DHCPv6 Server for CMTS Subscribers

Configure ISC Kea to handle CMTS subscriber prefix delegation:

```json
{
  "Dhcp6": {
    "interfaces-config": {
      "interfaces": ["eth1"]
    },
    "subnet6": [
      {
        "id": 1,
        "subnet": "2001:db8:cable::/40",
        "relay": {
          "ip-addresses": ["2001:db8:cmts:1::1"]
        },
        "pools": [
          {
            "pool": "2001:db8:cable::100-2001:db8:cable::ffff"
          }
        ],
        "pd-pools": [
          {
            "prefix": "2001:db8:cable::",
            "prefix-len": 40,
            "delegated-len": 56
          }
        ]
      }
    ]
  }
}
```

## Verifying Cable Modem IPv6 Status

```text
! Cisco CMTS - verify cable modems have IPv6
show cable modem ipv6

! Check specific modem IPv6 assignment
show cable modem 00aa.bbcc.ddee detail | include IPv6

! View DHCPv6 bindings relayed through CMTS
show ipv6 dhcp relay binding
```

## IPv6 Multicast on CMTS

Cable TV and IPTV over IPv6 multicast requires MLD (Multicast Listener Discovery) on the CMTS:

```text
! Enable MLD on cable interface
interface Cable1/0/0
 ipv6 mld join-group ff02::1
 ipv6 mld version 2
 ipv6 mld query-interval 125
```

## Conclusion

Configuring IPv6 on a CMTS involves enabling IPv6 on cable interfaces, setting up DHCPv6 relay to a prefix delegation server, and ensuring DOCSIS configuration files enable IPv6 on cable modems. With DOCSIS 3.1's wide deployment, most modern cable infrastructure supports IPv6 natively.
