# How to Debug Router Advertisement Issues with rdisc6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Router Advertisement, Rdisc6, Debugging, Networking, Troubleshooting

Description: Use rdisc6 and related tools to debug IPv6 Router Advertisement issues, diagnose missing RA responses, and verify prefix, DNS, and routing information delivery.

## Introduction

`rdisc6` is a command-line tool from the `ndisc6` package that sends an IPv6 Router Solicitation and displays the Router Advertisement response in a human-readable format. It is the primary tool for debugging RA delivery issues.

## Installing rdisc6

```bash
# Debian/Ubuntu

sudo apt-get install ndisc6

# RHEL/CentOS/Fedora
sudo dnf install ndisc6

# Arch Linux
sudo pacman -S ndisc6
```

## Basic rdisc6 Usage

```bash
# Send a Router Solicitation on eth0 and display the RA response
rdisc6 eth0

# Example output:
# Soliciting ff02::2 (ff02::2) on eth0...
#
# Hop limit                 :           64 (      0x40)
# Stateful address conf.    :           No
# Stateful other conf.      :           No
# Mobile home agent         :           No
# Router preference         :       medium
# Neighbor discovery proxy  :           No
# Router lifetime           :         1800 (0x00000708) seconds
# Reachable time            :  unspecified (0x00000000)
# Retransmit time           :  unspecified (0x00000000)
#  Prefix                   : 2001:db8:1:1::/64
#   Valid time              :        86400 (0x00015180) seconds
#   Pref. time              :        14400 (0x00003840) seconds
# Recursive DNS server      : 2001:db8:1:1::53
#  DNS server lifetime      :          600 seconds
# DNS search list           : example.com
#  Search list lifetime     :          600 seconds
# Source link-layer address : 00:11:22:33:44:55
# from fe80::211:22ff:fe33:4455
```

## Common Debugging Scenarios

### Scenario 1: No RA Response Received

```bash
# Try with a longer timeout (default is 1 second)
rdisc6 -r 3 eth0
# -r retries: send the solicitation up to 3 times

rdisc6 -w 5000 eth0
# -w timeout: wait 5000ms for each response
```

If still no response, check on the router side:

```bash
# On the router: verify radvd is running
sudo systemctl status radvd

# Check if radvd is sending RAs at all
sudo tcpdump -i eth1 -v "icmp6 and ip6[40] == 134"
# Type 134 = Router Advertisement
```

### Scenario 2: RA Received But No Address Configured on Client

```bash
# Verify the AdvAutonomous flag is on in the RA
rdisc6 eth0 | grep -i "autonomous"
# Should show: Autonomous address conf. : Yes

# Check kernel is accepting RAs
sysctl net.ipv6.conf.eth0.accept_ra
# Must be 1 (or 2 for forwarding hosts)

# If accept_ra is 0, enable it
sudo sysctl -w net.ipv6.conf.eth0.accept_ra=1
```

### Scenario 3: No Default Route Installed

```bash
# Check the router lifetime in the RA
rdisc6 eth0 | grep "Router lifetime"
# If 0 seconds, this router is not advertising as a default gateway

# Verify the default route was installed
ip -6 route show default
# Expected: default via fe80::... dev eth0 proto ra
```

### Scenario 4: DNS Not Being Configured

```bash
# Check if RDNSS is in the RA
rdisc6 eth0 | grep -A2 "DNS"
# If absent, RDNSS is not configured in radvd

# On the router, verify RDNSS block in radvd.conf
grep -A3 "RDNSS" /etc/radvd.conf

# Check systemd-resolved on the client
systemd-resolve --status | grep "DNS Servers"
```

## Using tcpdump for RA Packet Capture

For detailed inspection of RA packets:

```bash
# Capture all ICMPv6 RA packets and save to file
sudo tcpdump -i eth0 -w /tmp/ra_capture.pcap "icmp6 and ip6[40] == 134"

# Analyze with verbose output
sudo tcpdump -r /tmp/ra_capture.pcap -v

# Filter for specific source (the router's link-local address)
sudo tcpdump -i eth0 -v "icmp6 and src fe80::1 and ip6[40] == 134"
```

## Using ip monitor to Watch RA Effects

```bash
# Watch real-time changes to IPv6 addresses (triggered by RA)
ip -6 monitor address &

# Force a Router Solicitation by cycling the interface
sudo ip link set eth0 down && sudo ip link set eth0 up

# Observe new addresses being added to the output
```

## Checking radvd Logs

```bash
# View radvd systemd journal logs
sudo journalctl -u radvd -f

# Enable debug logging in radvd.conf
# Add globally: debug 1;
# or start with: radvd -d 5 (debug level 5)
```

## Conclusion

`rdisc6` is the fastest way to verify what a router is actually advertising, making it invaluable for debugging SLAAC and RA configuration issues. When combined with `tcpdump` for packet capture and `ip monitor` for real-time event watching, you have a comprehensive toolkit for diagnosing any Router Advertisement problem from prefix misconfiguration to missing DNS options.
