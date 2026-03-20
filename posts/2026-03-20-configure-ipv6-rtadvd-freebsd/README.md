# How to Configure IPv6 Router Advertisements with rtadvd on FreeBSD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, FreeBSD, rtadvd, Router Advertisements, SLAAC

Description: Learn how to configure rtadvd on FreeBSD to send IPv6 Router Advertisements, enabling SLAAC on your network so clients can automatically configure IPv6 addresses.

## What is rtadvd?

`rtadvd` (Router Advertisement Daemon) is the FreeBSD daemon that sends ICMPv6 Router Advertisement (RA) messages. Client machines listen for these RAs to perform SLAAC — automatically assigning IPv6 addresses and default routes.

## Enable rtadvd

```bash
# Enable IPv6 forwarding (required for rtadvd)
sysctl -w net.inet6.ip6.forwarding=1
echo 'net.inet6.ip6.forwarding=1' >> /etc/sysctl.conf

# Enable rtadvd in rc.conf
cat >> /etc/rc.conf << 'EOF'
ipv6_gateway_enable="YES"
rtadvd_enable="YES"
rtadvd_interfaces="em1"    # Interface(s) to advertise on (downstream)
EOF

service rtadvd start
```

## Configure /etc/rtadvd.conf

```bash
# Create /etc/rtadvd.conf
cat > /etc/rtadvd.conf << 'EOF'
# em1: downstream LAN interface
em1:\
    :addrs#1:\
    :addr="2001:db8:lan::":\
    :prefixlen#64:\
    :pinfoflags="la":\
    :rltime#1800:\
    :AdvDefaultLifetime#1800:\
    :AdvReachableTime#0:\
    :AdvRetransTimer#0:\
    :AdvCurHopLimit#64:\
    :AdvLinkMTU#1500:\
    :mtu#1500:
EOF
```

## rtadvd.conf Fields Explained

```
em1:                    Interface name
:addrs#1:               Number of prefix entries
:addr="2001:db8:lan::": Prefix to advertise (::= unspecified last 64 bits)
:prefixlen#64:          Prefix length
:pinfoflags="la":       l=on-link flag, a=autonomous (SLAAC) flag
:rltime#1800:           Router lifetime (seconds, 0=not default router)
:AdvDefaultLifetime#1800: Default router lifetime
:AdvLinkMTU#1500:       MTU to advertise
```

## Configure DNS in Router Advertisements (RDNSS)

```bash
# Include DNS server in RA (RFC 8106)
cat > /etc/rtadvd.conf << 'EOF'
em1:\
    :addrs#1:\
    :addr="2001:db8:lan::":\
    :prefixlen#64:\
    :pinfoflags="la":\
    :rltime#1800:\
    :rdnss="2001:db8::1":\
    :rdnss_lifetime#3600:\
    :dnssl="example.com":\
    :dnssl_lifetime#3600:
EOF
```

## Multiple Prefixes on One Interface

```bash
cat > /etc/rtadvd.conf << 'EOF'
em1:\
    :addrs#2:\
    :addr="2001:db8:lan::":\
    :prefixlen#64:\
    :pinfoflags="la":\
    :addr="fd00:db8:lan::":\
    :prefixlen#64:\
    :pinfoflags="la":\
    :rltime#1800:
EOF

service rtadvd restart
```

## Verify rtadvd is Working

```bash
# Check rtadvd is running
service rtadvd status
pgrep rtadvd

# Send an immediate RA (SIGUSR1)
kill -USR1 $(pgrep rtadvd)

# Capture RAs being sent
tcpdump -i em1 -n 'icmp6 and ip6[40] == 134'
# Type 134 = Router Advertisement

# On a client, check if SLAAC address was assigned
# (on another machine connected to em1)
ifconfig em0 | grep inet6
```

## rtadvd for Stateless DHCPv6 (M and O flags)

```bash
# O flag: clients should get options from DHCPv6 (DNS, etc.)
# M flag: clients should get addresses from DHCPv6 (stateful)
cat > /etc/rtadvd.conf << 'EOF'
em1:\
    :addrs#1:\
    :addr="2001:db8:lan::":\
    :prefixlen#64:\
    :pinfoflags="la":\
    :rltime#1800:\
    :mflag#0:\
    :oflag#1:
EOF
# mflag=0, oflag=1: SLAAC addresses + DHCPv6 for DNS options
```

## Summary

Configure rtadvd on FreeBSD to send Router Advertisements by enabling `rtadvd_enable="YES"` and `ipv6_gateway_enable="YES"` in `/etc/rc.conf`, then create `/etc/rtadvd.conf` with the prefix, prefixlen, and flags. The `pinfoflags="la"` sets the on-link (`l`) and autonomous/SLAAC (`a`) flags. Include RDNSS for DNS-in-RA. Verify with `tcpdump -i em1 -n 'icmp6 and ip6[40] == 134'`.
