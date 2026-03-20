# How to Configure IPv6 for CMTS Equipment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, CMTS, Cable, DOCSIS, ISP, DHCPv6

Description: Configure IPv6 on Cable Modem Termination Systems (CMTS) for DOCSIS cable broadband subscribers including DHCPv6, prefix delegation, and CPE configuration.

## DOCSIS IPv6 Overview

DOCSIS 3.0+ supports native IPv6 for cable subscribers. The IPv6 provisioning flow:

1. Cable modem powers on, requests IPv6 via DHCPv6
2. CMTS relays DHCPv6 to provisioning server
3. Cable modem gets IPv6 address + config file URL
4. CPE router gets prefix delegation (/56 or /48) via DHCPv6-PD
5. Home devices get addresses via SLAAC from CPE

## Cisco CMTS: IPv6 Configuration

```text
! Cisco uBR or cBR-8 CMTS - IPv6 configuration

! Configure cable interface for IPv6
interface Cable1/0/1
  ipv6 address 2001:db8:cmts::1/64
  ipv6 nd ra-interval 60
  ipv6 nd prefix 2001:db8:cmts::/64

! DHCPv6 relay for cable subscribers
  ipv6 dhcp relay destination 2001:db8::dhcp
  ipv6 helper-address 2001:db8::dhcp

! Enable IPv6 subscriber management
  cable ipv6 source-verify
  cable dhcpv6-giaddr policy

! Cable bundle interface
interface Bundle1
  ipv6 address 2001:db8:mgmt::1/64
  ipv6 nd ra-interval 30
  cable ipv6 address 2001:db8:subs::/64

! Verify IPv6 cable modem leases
show cable modem ipv6
show ipv6 dhcp binding | head -20
```

## ISC Kea DHCPv6 for CMTS

```json
{
    "Dhcp6": {
        "interfaces-config": {
            "interfaces": ["cmts0"]
        },
        "subnet6": [
            {
                "id": 1,
                "subnet": "2001:db8:subs::/48",
                "relay": {
                    "ip-addresses": ["2001:db8:cmts::1"]
                },
                "pools": [
                    {"pool": "2001:db8:subs::1 - 2001:db8:subs::ffff"}
                ],
                "pd-pools": [
                    {
                        "prefix": "2001:db8:home::/40",
                        "prefix-len": 40,
                        "delegated-len": 56
                    }
                ]
            }
        ]
    }
}
```

## Monitoring Cable IPv6 Subscribers

```bash
# Check cable modem IPv6 registration

show cable modem ipv6 | grep "online"

# DHCPv6 statistics
show ipv6 dhcp statistics

# Subscriber counts
show cable modem summary | grep "IPv6"
```

## Conclusion

CMTS IPv6 configuration for DOCSIS networks requires enabling IPv6 on cable interfaces, configuring DHCPv6 relay to forward subscriber DHCPv6 requests to the provisioning server, and allocating per-subscriber /56 prefixes for home network delegation. Cisco CMTS platforms use `cable ipv6` commands for subscriber management. Monitor cable modem IPv6 registration with `show cable modem ipv6` and alert when IPv6 registration rates fall below IPv4 rates, indicating provisioning issues.
