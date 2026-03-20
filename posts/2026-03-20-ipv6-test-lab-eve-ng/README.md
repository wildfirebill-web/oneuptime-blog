# How to Build an IPv6 Test Lab in EVE-NG

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: EVE-NG, IPv6, Test Lab, Networking, Simulation, Cisco

Description: Build an IPv6 test lab in EVE-NG (Emulated Virtual Environment) with Cisco, Juniper, and open-source router images.

## EVE-NG Overview

EVE-NG (Emulated Virtual Environment - Next Generation) is a network emulation platform that runs router images as KVM virtual machines. It supports:
- Cisco IOS, IOS-XE, NX-OS, IOS-XR
- Juniper vSRX, vMX, vQFX
- Arista vEOS
- FRRouting (Linux-based, free)

## Adding FRR to EVE-NG (Free Option)

```bash
# On EVE-NG server, add FRR as a custom Linux image
# SSH to EVE-NG server
ssh root@eve-ng-server

# Download and install FRR appliance for EVE-NG
cd /opt/unetlab/addons/iol/bin/

# Create a startup script for FRR nodes
cat > /opt/unetlab/addons/qemu/linux-frr-startup.sh << 'EOF'
#!/bin/bash
# Configure interfaces and start FRR
ip link set eth0 up
ip link set eth1 up

# Start FRR daemons
/usr/lib/frr/frrinit.sh start
EOF
```

## IPv6 Lab Topology in EVE-NG

Create a lab file (`.unl`) via EVE-NG API:

```python
import requests
from requests.auth import HTTPBasicAuth

EVE_HOST = "http://eve-ng-server"
AUTH = HTTPBasicAuth("admin", "eve")

def create_lab(name):
    """Create a new EVE-NG lab"""
    resp = requests.post(
        f"{EVE_HOST}/api/labs",
        auth=AUTH,
        json={
            "name": name,
            "version": "1",
            "description": "IPv6 test lab",
        }
    )
    return resp.json()

def add_node(lab_path, name, template, image, ethernet=4):
    """Add a node to the lab"""
    resp = requests.post(
        f"{EVE_HOST}/api/labs/{lab_path}/nodes",
        auth=AUTH,
        json={
            "name": name,
            "template": template,
            "image": image,
            "left": 100,
            "top": 100,
            "ethernet": ethernet,
        }
    )
    return resp.json()

def add_network(lab_path, name, network_type="pnet0"):
    """Add a network (bridge)"""
    resp = requests.post(
        f"{EVE_HOST}/api/labs/{lab_path}/networks",
        auth=AUTH,
        json={
            "name": name,
            "type": network_type,
        }
    )
    return resp.json()
```

## Cisco IOS IPv6 Configuration in EVE-NG

```
! Cisco IOS/IOS-XE — OSPFv3 and BGP IPv6

! Enable IPv6 routing
ipv6 unicast-routing

! Configure interfaces
interface GigabitEthernet0/0
 no shutdown
 ipv6 address 2001:db8:12::1/64
 ipv6 ospf 1 area 0

interface Loopback0
 ipv6 address 2001:db8:1::1/128
 ipv6 ospf 1 area 0
 ipv6 ospf network point-to-point

! OSPFv3
ipv6 router ospf 1
 router-id 1.1.1.1
 log-adjacency-changes

! BGP with IPv6 address family
router bgp 65001
 bgp router-id 1.1.1.1
 no bgp default ipv4-unicast
 neighbor 2001:db8:12::2 remote-as 65002
 !
 address-family ipv6
  network 2001:db8:1::/48
  neighbor 2001:db8:12::2 activate
 exit-address-family
```

## Juniper vSRX IPv6 in EVE-NG

```
# Juniper Junos — OSPFv3 configuration
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:12::1/64
set interfaces lo0 unit 0 family inet6 address 2001:db8:1::1/128

set protocols ospf3 area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf3 area 0.0.0.0 interface lo0.0 passive

set routing-options autonomous-system 65001
set protocols bgp group EBGP type external
set protocols bgp group EBGP neighbor 2001:db8:12::2 peer-as 65002
set protocols bgp group EBGP family inet6 unicast
```

## EVE-NG Lab Validation Script

```bash
#!/bin/bash
# Run from EVE-NG host to validate IPv6 lab

NODES=("10.0.0.101" "10.0.0.102" "10.0.0.103")  # Management IPs

for NODE in "${NODES[@]}"; do
    echo "=== Validating ${NODE} ==="

    # Check IPv6 routing table
    ssh -o StrictHostKeyChecking=no admin@${NODE} \
        "show ipv6 route" 2>/dev/null | grep -c "O" | \
        xargs -I{} echo "  OSPFv3 routes: {}"

    # Check BGP peers
    ssh admin@${NODE} \
        "show bgp ipv6 unicast summary" 2>/dev/null | \
        grep "Established" | wc -l | \
        xargs -I{} echo "  BGP sessions up: {}"
done
```

## Conclusion

EVE-NG provides the most realistic IPv6 network emulation with actual vendor images. Free tiers support Linux-based nodes like FRR; commercial licenses add Cisco and Juniper image support. The EVE-NG API enables lab automation for rapid topology provisioning. Key IPv6 features to test in EVE-NG: OSPFv3, BGP4+ with IPv6 address families, DHCPv6, prefix delegation, and dual-stack service configurations. Use FRR nodes for cost-free IPv6 routing protocol testing.
