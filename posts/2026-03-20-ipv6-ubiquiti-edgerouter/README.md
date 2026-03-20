# How to Configure IPv6 on Ubiquiti EdgeRouter

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Ubiquiti, EdgeRouter, EdgeOS, DHCPv6, Networking

Description: Configure IPv6 on Ubiquiti EdgeRouter using EdgeOS with DHCPv6 prefix delegation, stateless address autoconfiguration, and IPv6 firewall rules.

## Introduction

Ubiquiti EdgeRouter runs EdgeOS, which is based on Vyatta (a Linux-based routing platform). IPv6 configuration on EdgeRouter uses the `set` command structure with a hierarchical configuration model similar to Junos.

## Step 1: Configure WAN Interface with DHCPv6-PD

```bash
# Connect to the EdgeRouter via SSH

# Configure DHCPv6 client on the WAN interface

configure

# Configure eth0 (WAN) for DHCPv6 with prefix delegation
set interfaces ethernet eth0 dhcpv6-pd rapid-commit enable
set interfaces ethernet eth0 dhcpv6-pd send-ia na 0
set interfaces ethernet eth0 dhcpv6-pd send-ia pd 0
set interfaces ethernet eth0 dhcpv6-pd pd 0 length 64
set interfaces ethernet eth0 dhcpv6-pd pd 0 prefix-length /56

# Assign the delegated prefix to the LAN interface (eth1)
set interfaces ethernet eth0 dhcpv6-pd pd 0 interface eth1 host-address ::1
set interfaces ethernet eth0 dhcpv6-pd pd 0 interface eth1 prefix-id :1
set interfaces ethernet eth0 dhcpv6-pd pd 0 interface eth1 service slaac
```

## Step 2: Configure Static IPv6 Address (Alternative)

If your ISP provides a static IPv6 prefix:

```bash
# Assign a static IPv6 address to the WAN interface
set interfaces ethernet eth0 address 2001:db8:isp::2/64

# Configure a default IPv6 route
set protocols static route6 ::/0 next-hop 2001:db8:isp::1

# Assign a /64 to the LAN interface
set interfaces ethernet eth1 address 2001:db8:1:1::1/64
```

## Step 3: Configure SLAAC via Router Advertisements

```bash
# Enable Router Advertisements on the LAN interface
set interfaces ethernet eth1 ipv6 router-advert send-advert true
set interfaces ethernet eth1 ipv6 router-advert max-interval 100
set interfaces ethernet eth1 ipv6 router-advert min-interval 30
set interfaces ethernet eth1 ipv6 router-advert link-mtu 0
set interfaces ethernet eth1 ipv6 router-advert managed-flag false
set interfaces ethernet eth1 ipv6 router-advert other-config-flag false
set interfaces ethernet eth1 ipv6 router-advert default-lifetime 1800

# Configure the prefix to advertise
set interfaces ethernet eth1 ipv6 router-advert prefix ::/64 autonomous-flag true
set interfaces ethernet eth1 ipv6 router-advert prefix ::/64 on-link-flag true
set interfaces ethernet eth1 ipv6 router-advert prefix ::/64 valid-lifetime 86400
set interfaces ethernet eth1 ipv6 router-advert prefix ::/64 preferred-lifetime 14400

# Advertise DNS server
set interfaces ethernet eth1 ipv6 router-advert dns-server 2606:4700:4700::1111

commit
save
```

## Step 4: Configure IPv6 Firewall

```bash
# Create IPv6 firewall ruleset for WAN inbound traffic
set firewall ipv6-name WAN_IN_V6 default-action drop
set firewall ipv6-name WAN_IN_V6 description "WAN to LAN IPv6"

# Allow established and related
set firewall ipv6-name WAN_IN_V6 rule 10 action accept
set firewall ipv6-name WAN_IN_V6 rule 10 description "Allow established/related"
set firewall ipv6-name WAN_IN_V6 rule 10 state established enable
set firewall ipv6-name WAN_IN_V6 rule 10 state related enable

# Allow ICMPv6 (required for IPv6 operation)
set firewall ipv6-name WAN_IN_V6 rule 20 action accept
set firewall ipv6-name WAN_IN_V6 rule 20 description "Allow ICMPv6"
set firewall ipv6-name WAN_IN_V6 rule 20 protocol icmpv6

# Apply to WAN interface
set interfaces ethernet eth0 firewall in ipv6-name WAN_IN_V6

commit
save
```

## Step 5: Verify Configuration

```bash
# Show IPv6 addresses
show interfaces detail | grep inet6

# Show IPv6 routing table
show ipv6 route

# Show DHCPv6 prefix delegation
show dhcpv6-pd detail

# Show IPv6 neighbors
show ipv6 neighbors

# Ping test
ping6 2606:4700:4700::1111 count 3
```

## GUI Alternative (Wizard)

For a quick setup via the web interface:

1. Navigate to **Wizards > Basic Setup**
2. In the IPv6 section, select **DHCPv6** for WAN and **Auto** for LAN
3. The wizard configures prefix delegation and RA automatically

For more advanced settings, use the **Config Tree** tab in the web UI which provides GUI access to the full `set` command hierarchy.

## Conclusion

Ubiquiti EdgeRouter's EdgeOS provides a clean IPv6 configuration experience through its hierarchical `set` command structure. DHCPv6-PD is the standard way to obtain a prefix from residential ISPs, and the integrated RA configuration on LAN interfaces enables SLAAC for clients. The IPv6 firewall mirrors the IPv4 firewall structure with named rulesets applied to interfaces.
