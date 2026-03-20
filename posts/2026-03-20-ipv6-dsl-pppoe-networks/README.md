# How to Configure IPv6 for DSL/PPPoE Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DSL, PPPoE, DSLAM, ISP, Prefix Delegation

Description: Configure IPv6 for DSL networks using PPPoEv6 with prefix delegation, including DSLAM configuration and subscriber address management.

## IPv6 Over PPPoE Architecture

PPPoEv6 extends the PPPoE protocol to carry IPv6. DSL subscribers negotiate a PPPoEv6 session and receive:
- A /64 link-local address for the PPP link
- A /128 or /64 for the WAN interface
- A /56 or /48 delegated prefix for the LAN

## PPPoE Server (LNS) Configuration on Linux

Use `accel-ppp` as a PPPoE server with IPv6 support:

```text
# /etc/accel-ppp.conf

[modules]
log_file
pppoe
auth_mschap_v2
radius
ipv6pool
ipv6_dhcp

[core]
thread-count=4

[ppp]
ipv4=require
ipv6=require

[pppoe]
interface=eth1    # DSL-facing interface

[ipv6pool]
# Pool of /56 prefixes to delegate

delegate=2001:db8:dsl::/40,56

[ipv6-dns]
dns1=2001:db8:dns::1
dns2=2001:db8:dns::2

[radius]
server=2001:db8:radius::10,secret123
auth-port=1812
acct-port=1813
```

Start accel-ppp:

```bash
systemctl enable accel-ppp
systemctl start accel-ppp
```

## Customer Router (CPE) PPPoEv6 Configuration

On the customer router running OpenWRT:

```text
# /etc/config/network

config interface 'wan'
    option ifname   'eth0.2'
    option proto    'pppoe'
    option username 'user@isp.com'
    option password 'password'
    option ipv6     '1'

config interface 'wan6'
    option ifname   '@wan'
    option proto    'dhcpv6'
    option reqprefix '56'

config interface 'lan'
    option ifname   'br-lan'
    option proto    'static'
    option ip6assign '64'
```

## RADIUS for IPv6 Attribute Assignment

FreeRADIUS returns IPv6 prefix assignment for each subscriber:

```text
# /etc/freeradius/3.0/users
user@isp.com Cleartext-Password := "password"
    Framed-Protocol = PPP,
    Framed-IPv6-Prefix = "2001:db8:dsl:1234::/64",
    Delegated-IPv6-Prefix = "2001:db8:dsl:1234::/56"
```

## DSLAM IPv6 Configuration

Configure the DSLAM (Digital Subscriber Line Access Multiplexer) to pass IPv6 traffic:

```nginx
! Huawei MA5800 DSLAM
! Enable IPv6 on the upstream port
interface GigabitEthernet 0/0/0
  ipv6 enable
  ipv6 address 2001:db8:dslam::1/64

! Enable DHCPv6 snooping to pass PD requests
dhcpv6 snooping enable
dhcpv6 snooping information enable
```

## Monitoring PPPoEv6 Sessions

```bash
# On accel-ppp server: list active IPv6 sessions
accel-cmd show sessions | grep -E "ipv6|pppoe"

# Show specific session detail
accel-cmd show sessions username user@isp.com
```

## Common Issues

- **PPPoEv6 session negotiates but no IPv6 address**: Check that RADIUS returns the correct IPv6 attributes and that the PPPoE server has IPv6 pools configured.
- **Prefix delegation not working**: Ensure the CPE is requesting PD (check `reqprefix` option) and the server has PD pools available.
- **IPv6 connectivity but no DNS**: Verify DNS attributes are being returned in DHCPv6 or RA messages.

## Conclusion

IPv6 on DSL/PPPoE networks uses PPPoEv6 for session establishment and DHCPv6-PD for prefix delegation. The `accel-ppp` server on Linux handles this cleanly with RADIUS integration for per-subscriber configuration. Verify RADIUS attribute support on your AAA server to ensure proper IPv6 prefix delivery.
