# How to Configure DHCP Option 82 (Relay Agent Information)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Networking, Option 82, DHCP Relay, Security, sysadmin

Description: DHCP Option 82 allows relay agents to insert circuit-ID and remote-ID sub-options into DHCP requests, enabling the DHCP server to make policy decisions based on which switch port or access point the client is connected to.

## What Is Option 82?

Option 82 (Relay Agent Information Option, RFC 3046) lets the relay agent add metadata to DHCP requests:

- **Sub-option 1 (Circuit-ID)**: Identifies the relay agent port/circuit (e.g., `GigabitEthernet0/1`)
- **Sub-option 2 (Remote-ID)**: Identifies the relay agent itself (e.g., switch MAC or hostname)

The DHCP server can use this metadata to assign IPs from specific pools or apply policies per port.

## Cisco IOS: Enabling Option 82

On the relay agent (Layer-3 switch or router):
```
! Enable Option 82 insertion on the relay agent
ip dhcp relay information option

! Optional: trust Option 82 from downstream (for daisy-chained relays)
interface Vlan10
  ip address 10.0.10.1 255.255.255.0
  ip helper-address 10.0.0.53
  ip dhcp relay information trusted
```

## ISC dhcpd: Using Option 82 Data in Policies

The DHCP server can match on relay agent sub-options:

```
# /etc/dhcp/dhcpd.conf

# Match on Circuit-ID to assign IPs from specific ranges
class "floor-1-ports" {
    match if option agent.circuit-id = "GigabitEthernet0/1";
}

class "floor-2-ports" {
    match if option agent.circuit-id = "GigabitEthernet0/2";
}

subnet 10.0.10.0 netmask 255.255.255.0 {
    pool {
        allow members of "floor-1-ports";
        range 10.0.10.100 10.0.10.149;
    }
    pool {
        allow members of "floor-2-ports";
        range 10.0.10.150 10.0.10.199;
    }
    option routers 10.0.10.1;
}
```

## dnsmasq: Logging Option 82 Data

dnsmasq logs the relay agent info when Option 82 is present:

```
# /etc/dnsmasq.conf
log-dhcp
# Option 82 data appears in DHCP logs with "relay-agent-info" tag
```

## Capturing Option 82 with tcpdump/tshark

```bash
# Capture DHCP relayed packets
sudo tcpdump -i eth0 'port 67' -w /tmp/dhcp_relay.pcap

# Decode Option 82 in tshark
tshark -r /tmp/dhcp_relay.pcap -Y "bootp.option.type == 82" \
  -T fields -e bootp.option.agent_info_circuit_id \
             -e bootp.option.agent_info_remote_id
```

## Security Uses of Option 82

- **DHCP snooping integration**: Switches insert Option 82 on trusted uplinks; DHCP server can verify.
- **IP-to-port tracking**: Know exactly which switch port a leased IP is connected to.
- **Policy enforcement**: Different VLANs or floor policies based on circuit location.

## Key Takeaways

- Option 82 adds circuit-ID (port) and remote-ID (switch) metadata to DHCP requests.
- Configure `ip dhcp relay information option` on Cisco relay agents.
- ISC dhcpd uses `option agent.circuit-id` in class match statements.
- Option 82 enables per-port IP pool assignment and enhances DHCP audit trails.
