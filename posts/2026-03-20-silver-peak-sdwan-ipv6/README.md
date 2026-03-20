# How to Configure Silver Peak SD-WAN with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Silver Peak, SD-WAN, HPE Aruba, EdgeConnect, Orchestrator, WAN

Description: Configure IPv6 in Silver Peak (now HPE Aruba) SD-WAN including EdgeConnect appliance IPv6 interface setup, Orchestrator policy configuration, and IPv6 path quality monitoring.

---

Silver Peak SD-WAN, now HPE Aruba EdgeConnect, supports IPv6 on LAN and WAN interfaces. EdgeConnect appliances managed by the Orchestrator can route IPv6 traffic across overlay tunnels, apply business intent overlays to IPv6 flows, and monitor IPv6 path quality.

## EdgeConnect IPv6 Interface Configuration

```text
HPE Aruba Orchestrator:
Navigate to: Configuration > Appliances > [Appliance Name] > Deployment

LAN Interface Configuration:
  Interface: lan0
  IPv4: 192.168.1.1/24
  IPv6: 2001:db8:site-a::1/64 (Static)
  IPv6 DHCPv6 Server: Enabled
    Pool: 2001:db8:site-a::100 to ::ffff
    DNS: 2001:4860:4860::8888
  Router Advertisement: Enabled
    Interval: 30s
    Prefix: 2001:db8:site-a::/64
    Autonomous: Yes

WAN Interface Configuration:
  Interface: wan0
  IPv4: Dynamic (DHCP from ISP)
  IPv6: Dynamic (DHCPv6 from ISP)
  Label: MPLS or Broadband
```

## EdgeConnect Local Configuration

```bash
# SSH into EdgeConnect appliance

ssh admin@edgeconnect-ip

# Show IPv6 interface status
show interfaces ipv6

# Configure IPv6 on LAN interface
configure interface lan0 ipv6 address 2001:db8:site-a::1/64

# Configure IPv6 default route
configure ipv6 route default-route 2001:db8:wan::gateway wan0

# Show IPv6 routing table
show ip route ipv6

# Verify SD-WAN tunnels (typically IPv4 underlay, IPv6 overlay)
show tunnels

# Test IPv6 connectivity
ping6 source-interface lan0 2001:4860:4860::8888
```

## Business Intent Overlay for IPv6

```text
Orchestrator > Configuration > Business Intent Overlays

Create Overlay: "IPv6-VoIP-Overlay"
  Application Definition:
    Match: IPv6
    Protocol: UDP
    Destination Port: 10000-20000
    DSCP: EF
  Path Quality:
    Latency: < 100ms
    Loss: < 1%
    Jitter: < 20ms
  Bonding:
    Mode: Highest Bandwidth
    WAN Links: MPLS, Broadband

Create Overlay: "IPv6-Bulk-Overlay"
  Application Definition:
    Match: IPv6
    Protocol: TCP
    Destination: All
  Path Quality:
    Loss: < 5%
  Bonding:
    Mode: Load Balance
```

## IPv6 Tunnel Configuration

```bash
# Silver Peak can create IPv6-in-IPv4 or IPv6-in-IPv6 tunnels

# Check tunnel endpoints
show tunnels detail | grep -i ipv6

# Configure tunnel with IPv6 endpoint
# In Orchestrator UI:
# Configuration > Topology > [Site] > Tunnels
# WAN IP: 2001:db8::edgeconnect (if ISP provides IPv6 WAN)
# Local WAN Label: Broadband-IPv6

# Silver Peak uses UDP 4163 for tunnel encapsulation
# Works over both IPv4 and IPv6 WAN connections
```

## Monitor IPv6 Traffic in Orchestrator

```python
#!/usr/bin/env python3
# silverpeak_ipv6_monitor.py - Monitor IPv6 via Silver Peak API

import requests

ORCH_URL = "https://orchestrator.example.com"
USERNAME = "admin"
PASSWORD = "password"

def get_session():
    """Authenticate to Orchestrator."""
    resp = requests.post(
        f"{ORCH_URL}/gms/rest/authentication/login",
        json={"user": USERNAME, "password": PASSWORD},
        verify=False
    )
    session = requests.Session()
    session.cookies.update(resp.cookies)
    return session

def get_ipv6_flows(session, appliance_id):
    """Get active IPv6 flows."""
    resp = session.get(
        f"{ORCH_URL}/gms/rest/flows/appliances/{appliance_id}",
        params={"ipVersion": "IPv6", "maxCount": 100}
    )
    return resp.json()

def get_tunnel_stats(session, appliance_id):
    """Get tunnel statistics including IPv6."""
    resp = session.get(
        f"{ORCH_URL}/gms/rest/stats/appliances/{appliance_id}/tunnels"
    )
    tunnels = resp.json()
    # Filter IPv6 tunnels
    ipv6_tunnels = [t for t in tunnels if ':' in t.get('remoteIp', '')]
    return ipv6_tunnels

if __name__ == '__main__':
    session = get_session()
    appliance_id = "appliance-id"

    flows = get_ipv6_flows(session, appliance_id)
    print(f"Active IPv6 flows: {len(flows)}")

    for flow in flows[:10]:
        print(f"  {flow.get('srcIpv6')} → {flow.get('dstIpv6')} "
              f"App: {flow.get('application')} BW: {flow.get('bandwidth')}Kbps")
```

## Quality of Service for IPv6

```bash
# EdgeConnect QoS for IPv6 traffic
# Configure in Orchestrator: Configuration > QoS

# QoS classes for IPv6:
# Class EF: VoIP RTP (DSCP EF, port 10000-20000)
# Class AF41: Video (DSCP AF41)
# Class CS3: SIP signaling (DSCP CS3)
# Class BE: Best effort

# Apply QoS policy via CLI
configure qos map-profile "IPv6-VoIP" \
    priority-queue 1 dscp ef
configure qos map-profile "IPv6-Video" \
    priority-queue 2 dscp af41

# Verify QoS statistics
show qos stats interface lan0 | grep "IPv6\|EF\|AF41"
```

HPE Aruba EdgeConnect SD-WAN IPv6 deployment centers on configuring dual-stack LAN interfaces with DHCPv6 for client addressing, defining Business Intent Overlays that match IPv6 flows for intelligent path selection, and leveraging the Orchestrator's centralized policy management to ensure consistent IPv6 QoS and routing across all branch sites.
