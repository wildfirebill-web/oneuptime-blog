# How to Configure a Router to Send SLAAC Router Advertisements

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SLAAC, Router Advertisement, IPv6, radvd, Cisco, Router Configuration

Description: Configure IPv6 routers to send Router Advertisements with prefix information for SLAAC, including Cisco IOS, Linux radvd, and key RA parameters for address autoconfiguration.

## Introduction

For SLAAC to work, the router must send Router Advertisements containing Prefix Information options with the A (autonomous) flag set. The router advertises the subnet prefix, lifetimes, and default gateway information. Hosts use this to generate their own addresses. This guide covers configuring RA for SLAAC on Cisco IOS routers and Linux using radvd.

## Cisco IOS SLAAC Configuration

```
! Step 1: Enable IPv6 routing
ipv6 unicast-routing

! Step 2: Configure interface with IPv6 address and prefix
interface GigabitEthernet0/1
 description LAN interface - SLAAC subnet
 ipv6 address 2001:db8::/64 eui-64
 ! Or use a specific address:
 ipv6 address 2001:db8::1/64
 ipv6 nd prefix 2001:db8::/64
 no shutdown

! Step 3: Configure RA parameters (optional, these are defaults)
interface GigabitEthernet0/1
 ! M flag = 0: don't use stateful DHCPv6 for addresses
 ! (default is 0, which enables SLAAC)
 no ipv6 nd managed-config-flag

 ! O flag = 0: don't use DHCPv6 for other config
 ! Set to 1 if you want DNS via DHCPv6 alongside SLAAC
 ! ipv6 nd other-config-flag

 ! Router Lifetime (seconds, 0 = not a default router)
 ! Default: 1800 seconds
 ipv6 nd ra-lifetime 1800

 ! RA interval (min and max in seconds)
 ! Default: send every 200 seconds
 ipv6 nd ra-interval 200

 ! Prefix lifetimes (valid, preferred in seconds)
 ! Default: valid=2592000 (30 days), preferred=604800 (7 days)
 ipv6 nd prefix 2001:db8::/64 2592000 604800

! Verify RA is being sent
show ipv6 interface GigabitEthernet0/1
! Look for: ND advertised reachable time, RA interval, prefix info
```

## Cisco IOS with Stateless DHCPv6 (O flag)

```
! SLAAC for address + DHCPv6 for DNS (common pattern)
interface GigabitEthernet0/1
 ipv6 address 2001:db8::1/64
 ! A=1 (SLAAC for addresses): default, no command needed
 ! Set O flag: clients use DHCPv6 for DNS server info
 ipv6 nd other-config-flag

! Configure stateless DHCPv6 pool for DNS
ipv6 dhcp pool STATELESS_POOL
 dns-server 2001:4860:4860::8888
 dns-server 2001:4860:4860::8844
 domain-name example.com

interface GigabitEthernet0/1
 ipv6 dhcp server STATELESS_POOL

! With this config:
! Hosts get address from SLAAC (RA prefix + EUI-64)
! Hosts get DNS from DHCPv6 (O flag triggers DHCPv6 INFO-REQUEST)
```

## Linux radvd Configuration

```bash
# Install radvd
sudo apt-get install radvd

# Create radvd configuration
cat > /etc/radvd.conf << 'EOF'
interface eth1 {
    # Send Router Advertisements
    AdvSendAdvert on;

    # Minimum seconds between RA (default: 200)
    MinRtrAdvInterval 200;
    MaxRtrAdvInterval 600;

    # Router Lifetime in RA (how long this router is valid as default gateway)
    AdvDefaultLifetime 1800;

    # M flag: 0 = use SLAAC for addresses (no stateful DHCPv6)
    AdvManagedFlag off;

    # O flag: 0 = no stateless DHCPv6 for other config
    # Set to on if you have a DHCPv6 server for DNS
    AdvOtherConfigFlag off;

    prefix 2001:db8::/64 {
        # L flag: prefix is on-link
        AdvOnLink on;

        # A flag: use this prefix for SLAAC (must be on!)
        AdvAutonomous on;

        # Valid lifetime (seconds, 0xffffffff = infinite)
        AdvValidLifetime 2592000;

        # Preferred lifetime
        AdvPreferredLifetime 604800;
    };
};
EOF

# Start radvd
sudo systemctl start radvd
sudo systemctl enable radvd

# Verify radvd is sending RAs
sudo radvdump  # Show RA content being sent
```

## radvd with RDNSS (DNS in RA)

```bash
# Configure DNS server in RA using RDNSS option (RFC 8106)
# This allows hosts to get DNS without DHCPv6

cat > /etc/radvd.conf << 'EOF'
interface eth1 {
    AdvSendAdvert on;
    MaxRtrAdvInterval 600;
    AdvDefaultLifetime 1800;
    AdvManagedFlag off;
    AdvOtherConfigFlag off;

    prefix 2001:db8::/64 {
        AdvOnLink on;
        AdvAutonomous on;
        AdvValidLifetime 2592000;
        AdvPreferredLifetime 604800;
    };

    # RDNSS: Include DNS servers in RA (no DHCPv6 needed)
    RDNSS 2001:4860:4860::8888 2001:4860:4860::8844 {
        AdvRDNSSLifetime 600;  # How long this DNS info is valid
    };

    # DNSSL: Include search domain in RA
    DNSSL example.com corp.example.com {
        AdvDNSSLLifetime 600;
    };
};
EOF

sudo systemctl restart radvd
```

## Key RA Parameters for SLAAC

```
RA Parameters Summary:

Router Lifetime:
  > 0: This router is a valid default gateway
  = 0: Prefix is advertised but router is NOT a default gateway
  Default: 1800 seconds (30 minutes)

Prefix Valid Lifetime:
  How long the address generated from this prefix remains valid
  When expires: address is INVALID (removed)
  Default: 2592000 seconds (30 days)
  Use 0xffffffff for "infinite"

Prefix Preferred Lifetime:
  When expires: address is DEPRECATED (existing connections OK, no new)
  Must be <= Valid Lifetime
  Default: 604800 seconds (7 days)

M flag (Managed):
  0 (off): Use SLAAC for address assignment (typical)
  1 (on): Use stateful DHCPv6 for address assignment

O flag (OtherConfig):
  0 (off): Use RA RDNSS for DNS or no DNS (typical with M=0)
  1 (on): Use stateless DHCPv6 for DNS/options

A flag (Autonomous):
  1 (on): SLAAC enabled for this prefix (must be 1 for SLAAC!)
  0 (off): Prefix advertised but not for SLAAC
  Configured per prefix, not per interface
```

## Conclusion

Enabling SLAAC requires the router to send RAs with Prefix Information options where the A flag is set. On Cisco IOS, this is the default behavior when `ipv6 unicast-routing` is enabled and an IPv6 address is configured on the interface. On Linux, radvd provides flexible RA configuration. The M=0/O=0 combination is pure SLAAC. M=0/O=1 adds DHCPv6 for DNS. Using RDNSS in the RA (radvd `RDNSS` block) eliminates the need for DHCPv6 entirely.
