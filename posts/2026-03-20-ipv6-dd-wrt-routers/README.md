# How to Configure IPv6 on DD-WRT Routers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DD-WRT, Router, Radvd, DHCPv6, Networking

Description: Configure IPv6 on DD-WRT routers using the web interface and SSH, enabling native IPv6 connectivity and SLAAC for connected devices.

## Introduction

DD-WRT is a popular open-source firmware for consumer routers. IPv6 support in DD-WRT is provided through native DHCPv6 client, radvd, and ip6tables. The configuration is accessible through the web interface (Administration > Management > IPv6) or via the CLI.

## Step 1: Enable IPv6 via Web Interface

1. Log in to the DD-WRT web interface (default: `http://192.168.1.1`)
2. Navigate to **Setup > IPv6**
3. Set **IPv6** to **Enable**
4. Set the **IPv6 Connection Type**:
   - **Native IPv6 from ISP**: Uses DHCPv6 stateless or stateful
   - **6in4 Tunnel**: For ISPs offering Hurricane Electric tunnel
   - **6to4 Tunnel**: Legacy transition mechanism
5. For **Native IPv6**:
   - Enable **DHCPv6**
   - Enable **Prefix Delegation** if your ISP delegates a prefix
6. Click **Save** and **Apply Settings**

## Step 2: Configure via SSH (Advanced)

SSH into the router for more control:

```bash
# Check current IPv6 configuration

ip -6 addr show

# Check if radvd is running
ps | grep radvd

# View the radvd configuration DD-WRT generated
cat /tmp/radvd.conf
```

## Step 3: Customize radvd via Startup Script

DD-WRT generates radvd.conf automatically, but you can customize via a startup script:

```bash
# Add to Administration > Commands > Startup
cat > /tmp/custom-radvd.conf << 'EOF'
interface br0 {
    AdvSendAdvert on;
    AdvManagedFlag off;
    AdvOtherConfigFlag off;
    MinRtrAdvInterval 30;
    MaxRtrAdvInterval 100;
    AdvDefaultLifetime 1800;

    prefix ::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvRouterAddr on;
        AdvValidLifetime 86400;
        AdvPreferredLifetime 14400;
    };

    RDNSS 2606:4700:4700::1111 2001:4860:4860::8888 {
        AdvRDNSSLifetime 600;
    };
};
EOF

# Restart radvd with custom config
killall radvd
radvd -C /tmp/custom-radvd.conf &
```

## Step 4: Configure IPv6 via NVRAM

For persistent configuration in DD-WRT, use NVRAM:

```bash
# Set IPv6 WAN type to DHCPv6
nvram set ipv6_proto=dhcp6

# Enable IPv6
nvram set ipv6_enable=1

# Enable prefix delegation
nvram set ipv6_prefix_delegation=1

# Set DNS servers
nvram set ipv6_dns=2606:4700:4700::1111 2001:4860:4860::8888

# Commit to flash
nvram commit

# Restart networking
service network restart
```

## Step 5: Configure IPv6 Firewall

```bash
# View current ip6tables rules
ip6tables -L -n

# Allow established/related connections
ip6tables -I FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow ICMPv6
ip6tables -I INPUT -p icmpv6 -j ACCEPT
ip6tables -I FORWARD -p icmpv6 -j ACCEPT

# Allow LAN to WAN
ip6tables -I FORWARD -i br0 -j ACCEPT

# Save rules using DD-WRT startup script mechanism
```

## Step 6: Verify IPv6 Connectivity

```bash
# Check WAN IPv6 address
ip -6 addr show vlan2  # or eth0 depending on router model

# Check LAN IPv6 address
ip -6 addr show br0

# Test outbound connectivity
ping6 2606:4700:4700::1111

# Check radvd is sending RAs
tcpdump -i br0 -v "icmp6 and ip6[40] == 134" -c 3
```

## Checking Client IPv6 Addresses

From a client device on the LAN:
```bash
# Linux
ip -6 addr show scope global

# Windows
ipconfig /all | findstr IPv6

# Test connectivity
ping -6 2606:4700:4700::1111

# Test DNS
nslookup -type=AAAA google.com
```

## DD-WRT Build Requirements

Not all DD-WRT builds include full IPv6 support. Ensure you have:
- A build that includes the `ipv6` and `radvd` packages
- For DHCPv6-PD support, ensure `odhcp6c` or `dhcp6c` is included
- Check the DD-WRT wiki for your router model's IPv6 support status

## Conclusion

DD-WRT provides IPv6 support through a combination of the web interface for basic configuration and SSH/startup scripts for advanced customization. The auto-generated radvd configuration handles most cases, but custom radvd configurations via startup scripts provide the flexibility needed for RDNSS, multiple prefixes, or specific timing requirements. Always verify that your DD-WRT build includes the necessary IPv6 packages before planning a deployment.
