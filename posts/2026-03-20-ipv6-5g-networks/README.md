# How IPv6 Works on 5G Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, 5G, Mobile Networks, 3GPP, PDU Session, NR, SA, NSA

Description: Understand IPv6 addressing in 5G standalone and non-standalone networks, PDU session types, 5G network slicing with IPv6, and how UE devices receive IPv6 addresses over 5G.

---

5G networks are designed as IPv6-first. 3GPP specifications mandate IPv6 support in 5G Standalone (SA) architecture, with PDU sessions supporting IPv4, IPv6, IPv4v6 (dual-stack), and Ethernet types. 5G's large address space requires IPv6 for efficient addressing at scale.

## 5G IPv6 Architecture

```
5G IPv6 Network Architecture:

UE (User Equipment)
└── 5G NR Radio (gNB)
    └── N2/N3 interfaces (IPv6)
        └── AMF (Access and Mobility Function)
        └── SMF (Session Management Function) - assigns IPv6 to UE
            └── UPF (User Plane Function)
                └── DN (Data Network) - Internet or private
                    └── IPv6 Internet

PDU Session Types:
- IPv4: Legacy IPv4 only
- IPv6: IPv6 only (preferred for 5G-native)
- IPv4v6: Dual-stack (most common today)
- Ethernet: L2 transport
- Unstructured: Non-IP traffic
```

## 5G IPv6 Address Assignment

```
How UE Gets IPv6 in 5G:

1. UE initiates PDU Session Establishment
2. SMF allocates IPv6 prefix (typically /64)
3. UPF configures uplink/downlink for the prefix
4. SMF sends Router Advertisement to UE via N1 interface
   - RA includes /64 prefix
   - UE performs SLAAC to form interface address
5. Optional: DHCPv6 for additional addresses

IPv6 Prefix Types in 5G:
- /64 per PDU session (standard SLAAC)
- /128 for specific UE address (DHCPv6 stateful)
- Multiple prefixes for multi-homed UEs

Typical 5G IPv6 addresses:
2001:db8:5g:ue1::1/64  (from SMF prefix delegation)
```

## 3GPP 5G IPv6 PDU Session

```
PDU Session Establishment (simplified):
UE → gNB → AMF → SMF
PDU Session Establishment Request:
  PDU Session Type: IPv6
  S-NSSAI: (network slice)

SMF → UPF:
  N4 Session Establishment Request
  IPv6 prefix: 2001:db8:5g::/64

SMF → UE (via AMF/gNB):
  PDU Session Establishment Accept
  IPv6 Address: 2001:db8:5g:ue1::/64
  PCO (Protocol Configuration Options):
    IPv6 DNS: 2001:4860:4860::8888
    MTU: 1500

UE sends Router Solicitation
SMF/UPF sends Router Advertisement with assigned /64
UE uses SLAAC to form: 2001:db8:5g:ue1::device-eui/64
```

## Verify IPv6 on 5G Device (Android/Linux)

```bash
# Android device (via adb)
adb shell

# Check 5G interfaces
ip link show
# Look for: rmnet_data0, wwan0, or similar mobile data interface

# Check IPv6 addresses
ip -6 addr show rmnet_data0
# Should show global IPv6 from 5G PDU session

# Check default route
ip -6 route show

# Test IPv6 connectivity
ping6 -I rmnet_data0 2606:4700:4700::1111

# Linux laptop with 5G modem (ModemManager)
nmcli device status  # Shows wwan0 state
nmcli connection show --active | grep "IPv6"

# Check ModemManager for IPv6 bearer info
mmcli -b /org/freedesktop/ModemManager1/Bearer/0
# Shows: ipv6.address, ipv6.prefix, ipv6.gateway, ipv6.dns
```

## 5G Network Slicing with IPv6

```
Network Slices and IPv6:
Each network slice can have different IPv6 addressing:

Slice 1 (eMBB - enhanced Mobile Broadband):
  PDU Session: IPv4v6 dual-stack
  IPv6 prefix: 2001:db8:slice1::/48
  Use: Consumer internet

Slice 2 (URLLC - Ultra-Reliable Low Latency):
  PDU Session: IPv6
  IPv6 prefix: 2001:db8:slice2::/48
  Use: Industrial IoT, autonomous vehicles

Slice 3 (mMTC - massive Machine Type Communication):
  PDU Session: IPv6 (massive IoT devices)
  IPv6 prefix: 2001:db8:slice3::/48
  Use: Smart city sensors, meters
```

## 5G IPv6 Configuration on Open5GS (Open Source 5G Core)

```yaml
# /etc/open5gs/smf.yaml - Session Management Function

smf:
  sbi:
    addr: 2001:db8::smf
    port: 7777

  pfcp:
    addr: 2001:db8::smf  # N4 interface to UPF

  gtpu:
    addr: 2001:db8::smf  # GTP-U endpoint

  subnet:
    # IPv6 pool for PDU sessions
    - addr: 2001:db8:ue::/48
      dnn: internet    # Data Network Name
    - addr: 10.45.0.0/16
      dnn: internet    # IPv4 pool

  dns:
    - 2001:4860:4860::8888   # IPv6 DNS for UEs
    - 8.8.8.8                # IPv4 DNS

  # MTU for UE sessions
  mtu: 1400
```

## Monitor 5G IPv6 Traffic

```bash
# On UPF (User Plane Function) - capture 5G user plane
sudo tcpdump -i ogstun -nn ip6

# Check Open5GS UPF IPv6 forwarding
sudo sysctl net.ipv6.conf.all.forwarding
# Must be 1

# Monitor active PDU sessions
curl http://127.0.0.100:7777/v1/sessions | python3 -m json.tool | grep -i ipv6

# Check IPv6 routes added by UPF for UE sessions
ip -6 route show table all | grep ogstun
```

5G networks treat IPv6 as the primary address family, with SMFs assigning /64 prefixes per PDU session and delivering them via Router Advertisement to UEs, enabling SLAAC configuration that scales to billions of 5G-connected IoT and mobile devices without NAT.
