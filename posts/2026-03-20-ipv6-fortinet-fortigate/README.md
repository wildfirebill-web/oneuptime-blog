# How to Configure IPv6 on Fortinet FortiGate

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Fortinet, FortiGate, Firewall, IPv6 Policy, Networking

Description: Configure IPv6 addressing, firewall policies, and DHCPv6 on Fortinet FortiGate firewalls for enterprise IPv6 deployments.

## Introduction

Fortinet FortiGate firewalls support full IPv6 including static and dynamic addressing, DHCPv6 server and relay, Router Advertisements, BGP/OSPFv3, and IPv6 security policies. IPv6 is configured through either the GUI or CLI.

## Step 1: Enable IPv6 on an Interface (CLI)

```bash
# Via FortiGate CLI
config system interface
    edit "wan1"
        set ip6 2001:db8:isp::2/64
        set allowaccess6 ping
    next
    edit "internal"
        config ipv6
            set ip6-address 2001:db8:1:1::1/64
            set ip6-allowaccess ping https ssh
            set ip6-send-adv enable
            set ip6-manage-flag disable
            set ip6-other-flag disable
            config ip6-prefix-list
                edit 2001:db8:1:1::/64
                    set autonomous-flag enable
                    set onlink-flag enable
                    set preferred-life-time 14400
                    set valid-life-time 86400
                next
            end
        end
    next
end
```

## Step 2: Configure IPv6 on Interface via GUI

1. Navigate to **Network > Interfaces**
2. Click on the interface to edit (e.g., `internal`)
3. Scroll to **IPv6** section
4. Set **IPv6 Address**: `2001:db8:1:1::1/64`
5. Enable **Send Advertisements**
6. Set **M Flag**: Off (for SLAAC)
7. Set **O Flag**: Off
8. Under **Prefix List**, add the /64 prefix with:
   - **Autonomous Flag**: On
   - **On-link Flag**: On
   - Valid/Preferred Lifetimes as desired
9. Click **OK**

## Step 3: Configure Static IPv6 Route

```bash
config router static6
    edit 1
        set dst ::/0
        set gateway 2001:db8:isp::1
        set device "wan1"
    next
end
```

## Step 4: Configure DHCPv6 Server

```bash
config system dhcp6 server
    edit 1
        set interface "internal"
        set ip-mode range
        set ip-range start-ip 2001:db8:1:1::1000 end-ip 2001:db8:1:1::1fff
        set dns-server1 2001:db8:1:1::53
        set dns-service local
        set domain "example.com"
        set lease-time 86400
    next
end
```

## Step 5: Configure IPv6 Firewall Policies

IPv6 policies are separate from IPv4 policies in FortiGate:

```bash
# Allow LAN IPv6 traffic to WAN
config firewall policy6
    edit 1
        set name "LAN-to-WAN-IPv6"
        set srcintf "internal"
        set dstintf "wan1"
        set srcaddr6 "all"
        set dstaddr6 "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat6 enable
        set logtraffic all
    next
end

# Allow established/related inbound (typically handled by stateful inspection)
# FortiGate's stateful firewall automatically handles return traffic
```

## Step 6: Configure IPv6 Firewall Address Objects

```bash
config firewall address6
    edit "Internal-IPv6-Network"
        set ip6 2001:db8:1:1::/64
    next
    edit "DNS-Servers-IPv6"
        set type range
        set start-ip 2001:db8:1:1::53
        set end-ip 2001:db8:1:1::53
    next
end
```

## Step 7: Configure OSPFv3 (Optional)

```bash
config router ospf6
    set router-id 1.1.1.1
    config area
        edit 0.0.0.0
        next
    end
    config ospf6-interface
        edit "internal"
            set interface "internal"
            set area 0.0.0.0
        next
    end
end
```

## Verification Commands

```bash
# Show IPv6 interface addresses
get system interface | grep -A3 ip6

# Show IPv6 routing table
get router info6 routing-table all

# Show DHCPv6 leases
get system dhcp6 lease

# Show IPv6 firewall policy statistics
get firewall policy6

# Ping test
execute ping6 2606:4700:4700::1111

# Traceroute
execute tracert6 2606:4700:4700::1111

# Check RA is being sent
diagnose ipv6 nd info
```

## Conclusion

FortiGate provides comprehensive IPv6 support with the interface-level IPv6 configuration containing both the address and RA settings. IPv6 security policies are managed separately from IPv4, allowing precise control over IPv6 traffic flows. For production deployments, configure both IPv4 and IPv6 policies in parallel and verify using the `diagnose ipv6` command family for deep inspection of the IPv6 stack.
