# How to Configure RA Guard on Juniper Switches

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RA Guard, Juniper, IPv6 Security, NDP Security, EX Series, Junos

Description: Configure IPv6 RA Guard on Juniper EX Series switches using Junos to protect host-facing ports from rogue Router Advertisement attacks.

## Introduction

Juniper EX Series switches implement RA Guard through the IPv6 Neighbor Discovery Inspection (NDI) feature in Junos OS. RA Guard is configured under the `[edit vlans]` or `[edit interfaces]` hierarchy, assigning trusted (router) and untrusted (host) port roles. This guide covers Junos configuration for EX Series switches running Junos 12.1 and later.

## RA Guard in Junos Architecture

Juniper implements RA Guard as part of the broader IPv6 ND security framework.

```text
Juniper RA Guard Framework:

Junos NDI Components:
  - RA Guard: blocks RA on untrusted ports
  - DHCPv6 snooping: tracks DHCPv6 bindings
  - ND snooping: builds neighbor table from NDP
  - IPv6 Source Guard: validates source based on bindings

Configuration Hierarchy:
  [edit vlans <vlan-name> forwarding-options]
    dhcp-security {        ← DHCPv6 snooping
      ...
    }
    nd-security {          ← RA Guard + ND inspection
      port-security;
    }

  [edit interfaces <interface> unit 0 family ethernet-switching]
    dhcp-trusted;          ← mark as router/DHCP server port
    nd-security-trusted;   ← mark as ND trusted port
```

## Basic RA Guard Configuration

Configure RA Guard at the VLAN level and mark trusted ports.

```text
# Step 1: Enable ND security on the VLAN

set vlans v10 forwarding-options nd-security

# Step 2: Mark trusted (router-facing) ports as ND-trusted
# All other ports are automatically untrusted (RA is blocked)
set interfaces ge-0/0/23 unit 0 family ethernet-switching nd-security-trusted

# Step 3: Commit the configuration
commit

# Verify: show vlans v10 detail
# Verify: show nd-security binding
```

## Full Configuration in Junos Hierarchy Format

```text
# Complete RA Guard configuration for VLAN 10

vlans {
    v10 {
        vlan-id 10;
        forwarding-options {
            nd-security {
                # Enable RA Guard on this VLAN
                # All host ports are untrusted by default
            }
        }
    }
}

interfaces {
    # Access ports (host-facing) - RA Guard enforced by default
    ge-0/0/0 {
        unit 0 {
            family ethernet-switching {
                interface-mode access;
                vlan {
                    members v10;
                }
                # No nd-security-trusted: RA will be blocked
            }
        }
    }

    # Uplink/router port - mark as trusted
    ge-0/0/23 {
        unit 0 {
            family ethernet-switching {
                interface-mode trunk;
                vlan {
                    members all;
                }
                nd-security-trusted;   # Allow RA through
            }
        }
    }
}
```

## Configuring RA Guard with DHCPv6 Guard Together

In production, deploy RA Guard alongside DHCPv6 snooping for complete first-hop protection.

```text
# Enable both ND security and DHCPv6 snooping on VLAN
set vlans v10 forwarding-options nd-security
set vlans v10 forwarding-options dhcp-security

# Mark uplink as trusted for both ND and DHCP
set interfaces ge-0/0/23 unit 0 family ethernet-switching nd-security-trusted
set interfaces ge-0/0/23 unit 0 family ethernet-switching dhcp-trusted

# The untrusted access ports are protected from:
# - Rogue Router Advertisements (RA Guard)
# - Rogue DHCPv6 servers (DHCPv6 Guard)
```

## Verification and Monitoring

Use Junos operational commands to verify RA Guard status and activity.

```bash
# Show ND security status on all VLANs
show nd-security

# Show ND security binding table
show nd-security binding

# Show ND security statistics (including dropped RA count)
show nd-security statistics

# Show configuration on a specific interface
show interfaces ge-0/0/0 detail | match "nd-security"

# Show VLAN ND security configuration
show vlans v10 detail

# Example output:
# VLAN: v10
#   ND security: Enabled
#   Trusted interfaces: ge-0/0/23
#   RA Guard statistics:
#     Received on untrusted ports: 12
#     Dropped: 12
#     Forwarded on trusted ports: 8
```

## Troubleshooting

```bash
# Check if RA is being dropped on host ports
show nd-security statistics interface ge-0/0/0

# Verify interface trust state
show nd-security interface ge-0/0/0
# Expected output for host port:
# Interface: ge-0/0/0
# Trust state: Untrusted
# RA Guard: Enabled

# Verify trusted port allows RA
show nd-security interface ge-0/0/23
# Expected output for router port:
# Interface: ge-0/0/23
# Trust state: Trusted
# RA Guard: Bypassed (trusted port)

# Common Issues:
#
# Issue 1: RA Guard dropping RA on router port
#   Fix: Add nd-security-trusted to the uplink interface
#   set interfaces ge-0/0/23 unit 0 family ethernet-switching nd-security-trusted
#
# Issue 2: nd-security not supported
#   Check Junos version: nd-security requires Junos 12.1 or later
#   show version
#
# Issue 3: RA Guard not enabled on VLAN
#   Verify: show vlans v10 detail | match nd-security
#   Fix: set vlans v10 forwarding-options nd-security
```

## Monitoring RA Guard with Syslog

```text
# Configure logging for ND security events
set system syslog file nd-security-log any info
set system syslog file nd-security-log structured-data

# Filter for ND security drops
# Log entries look like:
# RPD_ND_SECURITY_DROP: RA dropped on untrusted interface ge-0/0/1,
#   src fe80::bad:cafe, VLAN v10

# Monitor in real time
monitor log /var/log/nd-security-log
```

## Conclusion

Juniper EX Series switches implement RA Guard through the `nd-security` feature on VLANs. Enabling `nd-security` on a VLAN automatically enforces RA Guard on all ports in that VLAN. Router-facing ports are explicitly marked as trusted with `nd-security-trusted`. Combine with `dhcp-security` on the same VLAN for full first-hop security coverage. Verify with `show nd-security statistics` to confirm rogue RAs are being dropped.
