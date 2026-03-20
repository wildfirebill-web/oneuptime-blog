# How to Configure IPv6 First Hop Security on Juniper EX

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: First Hop Security, Juniper, IPv6 Security, EX Series, Junos, RA Guard

Description: Deploy IPv6 First Hop Security on Juniper EX Series switches using Junos, including RA Guard, DHCPv6 snooping, ND security, and IPv6 source guard configuration.

## Introduction

Juniper EX Series switches provide IPv6 First Hop Security through the IPv6 Neighbor Discovery (ND) Security and DHCPv6 snooping features in Junos. RA Guard and ND Inspection are part of the ND Security feature set, while DHCPv6 Guard uses the DHCPv6 snooping infrastructure. This guide covers complete FHS deployment on Juniper EX switches running Junos 12.1 or later.

## Juniper FHS Architecture

```
Juniper EX FHS Feature Mapping:

Cisco Feature          → Juniper Equivalent
RA Guard               → nd-security (drops RA on untrusted ports)
DHCPv6 Guard           → dhcp-security (v6) / dhcp-trusted
ND Inspection          → nd-security (validates NDP, builds bindings)
IPv6 Source Guard      → nd-security with binding enforcement
IPv6 Snooping          → nd-security (part of ND security)

Configuration Location:
  RA Guard + ND Inspection:
    [edit vlans <name> forwarding-options nd-security]
    [edit interfaces <if> unit 0 family ethernet-switching nd-security-trusted]

  DHCPv6 Snooping/Guard:
    [edit vlans <name> forwarding-options dhcp-security]
    [edit interfaces <if> unit 0 family ethernet-switching dhcp-trusted]
```

## Complete FHS Configuration

```
# Full Juniper EX FHS configuration for VLAN 10

# Step 1: Configure VLANs
set vlans v10 vlan-id 10
set vlans v20 vlan-id 20

# Step 2: Enable ND Security (RA Guard + ND Inspection) on VLANs
set vlans v10 forwarding-options nd-security
set vlans v20 forwarding-options nd-security

# Step 3: Enable DHCPv6 Snooping on VLANs
set vlans v10 forwarding-options dhcp-security
set vlans v20 forwarding-options dhcp-security

# Step 4: Set maximum bindings per interface (prevents exhaustion)
set vlans v10 forwarding-options nd-security maximum-bindings 10
set vlans v20 forwarding-options nd-security maximum-bindings 10

# Step 5: Configure access interfaces (host ports - untrusted by default)
set interfaces ge-0/0/0 unit 0 family ethernet-switching interface-mode access
set interfaces ge-0/0/0 unit 0 family ethernet-switching vlan members v10

set interfaces ge-0/0/1 unit 0 family ethernet-switching interface-mode access
set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members v10

# Step 6: Configure uplink (trusted port)
set interfaces ge-0/0/23 unit 0 family ethernet-switching interface-mode trunk
set interfaces ge-0/0/23 unit 0 family ethernet-switching vlan members all
# Mark as trusted for both ND and DHCP
set interfaces ge-0/0/23 unit 0 family ethernet-switching nd-security-trusted
set interfaces ge-0/0/23 unit 0 family ethernet-switching dhcp-trusted
```

## Junos Hierarchy Format Configuration

```
# Complete configuration in Junos bracket format

vlans {
    v10 {
        vlan-id 10;
        forwarding-options {
            nd-security {
                maximum-bindings 10;
            }
            dhcp-security {
                # DHCPv6 snooping enabled
            }
        }
    }
    v20 {
        vlan-id 20;
        forwarding-options {
            nd-security {
                maximum-bindings 10;
            }
            dhcp-security;
        }
    }
}

interfaces {
    # Access ports (ge-0/0/0 through ge-0/0/22) - untrusted
    ge-0/0/0 {
        unit 0 {
            family ethernet-switching {
                interface-mode access;
                vlan { members v10; }
                # No nd-security-trusted = RA Guard enforced
                # No dhcp-trusted = DHCPv6 server msgs blocked
            }
        }
    }

    # Uplink port - trusted
    ge-0/0/23 {
        unit 0 {
            family ethernet-switching {
                interface-mode trunk;
                vlan { members all; }
                nd-security-trusted;    # Allow RA through
                dhcp-trusted;           # Allow DHCPv6 server through
            }
        }
    }
}
```

## Verifying FHS on Juniper

```bash
# Verify ND security status
show nd-security

# Show binding table (equivalent to Cisco 'show ipv6 neighbor binding')
show nd-security binding

# Show ND security statistics per interface
show nd-security statistics

# Show per-interface security status
show nd-security interface ge-0/0/0
# Expected for host port:
# Interface: ge-0/0/0
# Trust State: Untrusted
# RA Guard: Enabled
# Max bindings: 10
# Current bindings: 2

show nd-security interface ge-0/0/23
# Expected for uplink:
# Interface: ge-0/0/23
# Trust State: Trusted
# RA Guard: Bypassed

# Check DHCPv6 snooping bindings
show dhcp v6 snooping binding

# Show DHCPv6 snooping statistics
show dhcp v6 snooping statistics
```

## Adding Static Bindings

For hosts that do not auto-learn (e.g., static address hosts), add manual bindings.

```
# Add static ND security binding
set vlans v10 forwarding-options nd-security static-binding
  inet6-address 2001:db8::1
  hardware-address 00:11:22:33:44:55
  interface ge-0/0/0

# In Junos bracket format:
vlans {
    v10 {
        forwarding-options {
            nd-security {
                static-binding 2001:db8::1 {
                    hardware-address 00:11:22:33:44:55;
                    interface ge-0/0/0;
                }
            }
        }
    }
}

# Verify static binding
show nd-security binding | match 2001:db8::1
```

## Troubleshooting

```bash
# Issue 1: RA Guard blocking uplink RA
# Symptom: Hosts not getting router advertisements
# Check:
show nd-security interface ge-0/0/23
# Fix: Ensure nd-security-trusted is set on uplink
set interfaces ge-0/0/23 unit 0 family ethernet-switching nd-security-trusted

# Issue 2: DHCPv6 clients not getting addresses
# Symptom: DHCPv6 ADVERTISE never reaches clients
# Check:
show dhcp v6 snooping statistics
# Fix: Add dhcp-trusted to DHCP server/relay port
set interfaces ge-0/0/23 unit 0 family ethernet-switching dhcp-trusted

# Issue 3: Binding table not populated
# Check:
show nd-security binding
# If empty: ensure nd-security is enabled on the VLAN
# Trigger re-population: have hosts send traffic (NDP will be seen)

# Issue 4: nd-security not available
# Check Junos version:
show version | match Junos
# Requires Junos 12.1 or later
# On older versions: use firewall filters for RA filtering
```

## Logging NDP Security Events

```
# Configure logging for ND security violations
set system syslog file fhs-log any info

# All ND security drops will appear in syslog:
# NDPMON_RA_DROP: RA dropped on untrusted port ge-0/0/1
# NDPMON_NA_SPOOF: NA spoofing detected, src fe80::bad, target 2001:db8::1
# NDPMON_BIND_EXCEEDED: Max bindings (10) exceeded on ge-0/0/5

# Monitor in real time
monitor log /var/log/fhs-log
```

## Conclusion

Juniper EX Series FHS is configured through `nd-security` and `dhcp-security` under the VLAN forwarding-options hierarchy. Enabling `nd-security` on a VLAN automatically enforces RA Guard and ND Inspection on all ports in that VLAN. Uplink and router ports must be explicitly marked trusted with `nd-security-trusted` and `dhcp-trusted`. Use `show nd-security binding` to verify the binding table and `show nd-security statistics` to confirm security features are active and dropping anomalous traffic.
