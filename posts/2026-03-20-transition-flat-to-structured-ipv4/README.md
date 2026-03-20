# How to Transition from Flat IPv4 Addressing to a Structured Subnet Design

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Migration, Network Design, Subnetting, VLAN, Network Restructuring

Description: Learn how to migrate a flat /8 or /16 network with everything in one subnet to a structured, hierarchical IPv4 design with proper VLAN segmentation.

## The Problem with Flat Networks

Many organizations start with everything in one large subnet:
- All servers, clients, printers, IoT on 192.168.1.0/24 or 10.0.0.0/8
- Excessive broadcast traffic as the network grows
- No security boundaries between device types
- Routing table is meaningless (everything "directly connected")

## Step 1: Document the Current State

```bash
# Discover all active devices on the flat network
nmap -sn 10.0.0.0/8 -oG - | awk '/Up$/{print $2}' > /tmp/all-hosts.txt

# Try to identify device types by port scan
nmap -p 22,80,443,8080,3389,8443 -oG /tmp/service-scan.txt 10.0.0.0/8

# Parse to identify:
# - Port 22: Servers, network devices
# - Port 3389: Windows desktops
# - Port 80/443: Web servers, management interfaces
# - Port 8080/8443: Cameras, IoT devices

cat /tmp/service-scan.txt | grep "22/open" | wc -l     # Likely servers
cat /tmp/service-scan.txt | grep "3389/open" | wc -l   # Windows devices
```

## Step 2: Design the Target State

```
Target: Segmented network with security zones

VLAN 10 — Corporate Users    10.1.10.0/24  (254 hosts)
VLAN 20 — Servers            10.1.20.0/24  (254 hosts)
VLAN 30 — DMZ                10.1.30.0/28  (14 hosts — small!)
VLAN 40 — Management/OOB     10.1.40.0/27  (30 network devices)
VLAN 50 — VoIP               10.1.50.0/24  (254 phones)
VLAN 60 — IoT                10.1.60.0/24  (isolated, no internet)
VLAN 99 — Native/Trunk       (untagged management traffic)

Migration from flat:
  Old: 10.0.0.0/8 (everything)
  New: 10.1.X.0/24 (structured)
```

## Step 3: Plan the Migration Sequence

Migrate in this order to minimize risk:

```
Phase 1: Network infrastructure (switches, routers, APs)
  - Move to VLAN 40 (management)
  - Low risk: these devices don't have users

Phase 2: Servers
  - Add secondary IPs in new subnet
  - Update DNS to new IPs
  - Move production traffic to new IPs
  - Remove old IPs after verification
  - Estimated downtime per server: 15 minutes

Phase 3: VoIP phones
  - Phones can be reconfigured centrally (DHCP option 66/150)
  - Move to VLAN 50

Phase 4: IoT/Cameras
  - Move to isolated VLAN 60
  - May require physical access for some devices

Phase 5: User workstations
  - Last — largest number of devices
  - Use DHCP to migrate (change pool to new subnet)
  - Short cutover window during non-business hours
```

## Step 4: Configure the New VLANs

```
! On the core switch
vlan 10
 name CORPORATE_USERS
vlan 20
 name SERVERS
vlan 30
 name DMZ
vlan 40
 name MANAGEMENT
vlan 50
 name VOIP
vlan 60
 name IOT

! Layer 3 switch (router-on-a-stick or L3 switch)
interface Vlan10
 ip address 10.1.10.1 255.255.255.0
 ip helper-address 10.1.20.10    ! DHCP server
!
interface Vlan20
 ip address 10.1.20.1 255.255.255.0
!
interface Vlan40
 ip address 10.1.40.1 255.255.255.224
```

## Step 5: Migrate a Server Without Downtime

```bash
# Phase 1: Add new IP as secondary
ip addr add 10.1.20.50/24 dev eth0

# Phase 2: Update DNS (low TTL)
# In DNS: server1.example.com A → 10.1.20.50 (TTL 60 seconds)

# Phase 3: Verify new IP is reachable
ping 10.1.20.50

# Phase 4: Monitor that clients are using new IP
tcpdump -i eth0 dst 10.0.0.50 -n | head   # Should decrease
tcpdump -i eth0 dst 10.1.20.50 -n | head  # Should increase

# Phase 5: Remove old IP (after all DNS TTLs expire)
ip addr del 10.0.0.50/8 dev eth0

# Phase 6: Update application configs, monitoring, etc.
```

## Step 6: Cut Over User DHCP

```bash
# Old DHCP pool (flat network)
# range 10.0.0.100 10.0.0.200;
# option routers 10.0.0.1;

# New DHCP pool (structured)
# range 10.1.10.100 10.1.10.200;
# option routers 10.1.10.1;

# Migration approach: change DHCP pool in maintenance window
# Users reconnect after lease expiry or reconnection
# Typically happens within 2-4 hours for all clients

# Force immediate renewal on Linux clients
sudo dhclient -r && sudo dhclient wlan0

# Force renewal on Windows
ipconfig /release && ipconfig /renew
```

## Conclusion

Transitioning from a flat network to a structured design requires planning the target VLAN/subnet architecture first, then migrating devices in phases: infrastructure first, then servers (using dual-IP during transition), then VoIP, IoT, and finally user workstations via DHCP pool cutover. The most critical step is maintaining the old IP as a secondary during server migration so services remain available throughout the transition.
