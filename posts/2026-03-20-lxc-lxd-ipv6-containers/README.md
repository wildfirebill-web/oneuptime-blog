# How to Configure LXC/LXD Containers with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: LXC, LXD, IPv6, Container Networking, Linux, System Containers

Description: A guide to configuring LXC and LXD system containers with IPv6 networking, including bridge configuration, SLAAC, DHCPv6, and per-container IPv6 address assignment.

LXD (the daemon managing LXC containers) provides full IPv6 support through its managed bridge networking. LXD containers behave like VMs from a networking perspective, making full dual-stack configuration straightforward.

## LXD Network Bridge with IPv6

```bash
# Check existing networks
lxc network list

# Create a new bridge with IPv6 enabled
lxc network create lxdbr1 \
  ipv4.address=10.100.0.1/24 \
  ipv4.nat=true \
  ipv6.address=fd00:lxd::/64 \
  ipv6.nat=true

# Inspect the network
lxc network show lxdbr1
```

## Enabling IPv6 on the Default Bridge

```bash
# Edit the default lxdbr0 network
lxc network set lxdbr0 ipv6.address fd00:lxd:default::/64
lxc network set lxdbr0 ipv6.nat true

# Verify
lxc network show lxdbr0 | grep ipv6
```

LXD automatically configures radvd or dnsmasq on the bridge to advertise the prefix via SLAAC, so containers receive IPv6 addresses without additional configuration.

## Launching Containers with IPv6

```bash
# Launch a container on the IPv6 bridge
lxc launch ubuntu:22.04 web1 --network lxdbr1

# Check container IP addresses
lxc exec web1 -- ip -6 addr show

# Or via lxc list
lxc list web1 --format json | python3 -m json.tool | grep -A 5 \"IPv6\"

# Access container via IPv6
lxc exec web1 -- curl -6 https://ipv6.google.com
```

## Static IPv6 Address for a Container

```bash
# Assign a static IPv6 address to a specific container
lxc network attach lxdbr1 web1 eth0

# Set static IPv6 address via device configuration
lxc config device set web1 eth0 ipv6.address fd00:lxd::10

# Verify
lxc exec web1 -- ip -6 addr show eth0
```

## LXD Profile with IPv6 Networking

```yaml
# Save as ipv6-profile.yaml
config: {}
description: Profile with IPv6 networking
devices:
  eth0:
    name: eth0
    network: lxdbr1
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: ipv6-profile
```

```bash
# Create the profile from YAML
lxc profile create ipv6-profile
lxc profile edit ipv6-profile < ipv6-profile.yaml

# Launch containers using the profile
lxc launch ubuntu:22.04 app1 --profile default --profile ipv6-profile

# Check applied profiles
lxc config show app1 | grep profiles
```

## LXD with Routed Networking (No NAT)

For production environments where containers should have globally routable IPv6:

```bash
# Configure a routed network (no NAT, containers get real prefixes)
lxc network create lxdrouted \
  ipv4.address=none \
  ipv6.address=2001:db8:lxd::/64 \
  ipv6.nat=false

# The host must have proper routing to forward to this prefix
# Containers receive addresses via SLAAC from the advertised prefix
lxc launch ubuntu:22.04 pub-web --network lxdrouted
lxc exec pub-web -- ip -6 addr show
```

## DHCPv6 Configuration in LXD

```bash
# Enable stateful DHCPv6 (in addition to SLAAC)
lxc network set lxdbr1 ipv6.dhcp true
lxc network set lxdbr1 ipv6.dhcp.stateful true

# Reserve a specific DHCPv6 address for a container
# (Uses the container's DUID — check with lxc info <container>)
lxc network set lxdbr1 ipv6.dhcp.ranges fd00:lxd::100-fd00:lxd::200
```

## IPv6 Firewall Rules for LXD Containers

LXD uses nftables to manage container traffic. Verify ICMPv6 is not blocked:

```bash
# Check nftables rules affecting lxdbr1
nft list ruleset | grep -A 5 lxdbr1

# Ensure ICMPv6 is allowed for NDP (essential for SLAAC)
ip6tables -L FORWARD -n | grep icmpv6

# Add ICMPv6 allow rule if missing
ip6tables -A FORWARD -i lxdbr1 -p ipv6-icmp -j ACCEPT
ip6tables -A FORWARD -o lxdbr1 -p ipv6-icmp -j ACCEPT
```

## Verifying IPv6 Connectivity

```bash
# Test IPv6 from container to internet
lxc exec web1 -- ping6 -c 3 2001:4860:4860::8888

# Test container-to-container IPv6
lxc exec web1 -- ping6 -c 3 fd00:lxd::app1-address

# Check SLAAC assignment worked
lxc exec web1 -- ip -6 addr show | grep "scope global"

# Verify default IPv6 route
lxc exec web1 -- ip -6 route show default
```

## Troubleshooting LXD IPv6

```bash
# If containers don't get IPv6 addresses, check radvd on host
lxc network show lxdbr1 | grep ipv6

# Restart the network
lxc network edit lxdbr1   # confirm settings are correct

# Check dnsmasq log for IPv6 SLAAC/DHCPv6
journalctl -u snap.lxd.daemon | grep -i "ipv6\|dhcp6\|radvd"

# Verify kernel IPv6 forwarding is enabled
sysctl net.ipv6.conf.all.forwarding
# Should be 1
```

LXD's managed networking makes IPv6 configuration straightforward: enable IPv6 on the bridge network and containers automatically receive SLAAC addresses. For production use, configure routed networking without NAT to give containers real globally routable IPv6 addresses.
