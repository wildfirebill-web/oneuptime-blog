# How to Monitor BGP Session Health with SNMP Traps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: BGP, SNMP, Monitoring, Traps, Cisco IOS, Network Management

Description: Learn how to configure SNMP traps for BGP session state changes on Cisco IOS devices to receive real-time alerts when BGP neighbors go up or down.

## Why SNMP Traps for BGP?

BGP session failures can be silent-traffic stops flowing but no alarm fires unless you have monitoring in place. SNMP traps send notifications to your NMS (Network Management System) the moment a BGP session changes state. This gives you near-real-time alerting without continuous polling.

## Relevant BGP SNMP MIBs

The BGP4-MIB (RFC 4273) defines objects for BGP monitoring:
- `bgpEstablished` trap: Session moved to Established state
- `bgpBackwardTransition` trap: Session moved away from Established (session down)
- `bgpPeerState` OID: Current state of each peer (1=Idle, 2=Connect, 3=Active, 4=OpenSent, 5=OpenConfirm, 6=Established)

## Step 1: Configure SNMP on the Router

First, configure SNMPv2c or SNMPv3 on the Cisco IOS device:

```text
! Configure SNMPv2c
Router(config)# snmp-server community public RO
Router(config)# snmp-server community private RW

! Define the SNMP trap receiver (your NMS server)
Router(config)# snmp-server host 192.168.1.100 version 2c public

! Set the contact and location for identification
Router(config)# snmp-server contact netops@example.com
Router(config)# snmp-server location "DC1-CoreRouter-Rack42"
```

## Step 2: Enable BGP SNMP Traps

Enable BGP-specific traps:

```text
! Enable BGP traps for session state changes
Router(config)# snmp-server enable traps bgp

! This enables both:
! - bgpEstablished (session comes up)
! - bgpBackwardTransition (session goes down)
```

Verify traps are enabled:

```text
Router# show snmp

! Output should include:
! BGP:                Enabled
```

## Step 3: Configure Trap Source Interface

Set the source interface for SNMP traps so the NMS can identify the router:

```text
! Use the loopback as the source for consistent identification
Router(config)# snmp-server trap-source Loopback0

! Set the engine ID for SNMPv3 (optional but recommended)
Router(config)# snmp-server engineID local 0102030405060708
```

## Step 4: Poll BGP Peer State via SNMP

Use `snmpwalk` to poll BGP peer states from your NMS:

```bash
# Walk the bgpPeerTable to get all neighbor states

snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.15.3

# Get peer state for a specific neighbor (X.X.X.X = neighbor IP as OID)
# For neighbor 203.0.113.1, the OID suffix is .203.0.113.1
snmpget -v2c -c public 192.168.1.1 \
  BGP4-MIB::bgpPeerState.203.0.113.1

# State values: 1=idle, 2=connect, 3=active, 4=opensent, 5=openconfirm, 6=established
```

## Step 5: Set Up SNMP Trap Receiver with snmptrapd

On your Linux monitoring server, configure snmptrapd to receive and log BGP traps:

```bash
# Install net-snmp tools
sudo apt-get install snmptrapd

# Edit /etc/snmp/snmptrapd.conf
cat > /etc/snmp/snmptrapd.conf << 'EOF'
# Accept traps from network devices
authCommunity log,execute,net public

# Log BGP traps to a file
traphandle BGP4-MIB::bgpEstablished /usr/local/bin/bgp_session_up.sh
traphandle BGP4-MIB::bgpBackwardTransition /usr/local/bin/bgp_session_down.sh
EOF

sudo systemctl enable snmptrapd
sudo systemctl start snmptrapd
```

## Step 6: Create a Trap Handler Script

Write a script that sends alerts when BGP sessions go down:

```bash
#!/bin/bash
# bgp_session_down.sh - Called by snmptrapd when BGP session drops

# Parse trap data from stdin
TRAP_DATA=$(cat)
ROUTER_IP=$(echo "$TRAP_DATA" | grep "^UDP:" | awk '{print $2}' | cut -d[ -f2 | cut -d] -f1)
PEER_IP=$(echo "$TRAP_DATA" | grep "bgpPeerRemoteAddr" | awk '{print $NF}')
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

# Log the event
echo "${TIMESTAMP} BGP SESSION DOWN: Router=${ROUTER_IP}, Peer=${PEER_IP}" >> /var/log/bgp_events.log

# Send alert via email or webhook
curl -s -X POST "https://your-alerting-system.example.com/webhook" \
  -H "Content-Type: application/json" \
  -d "{\"alert\": \"BGP session down\", \"router\": \"${ROUTER_IP}\", \"peer\": \"${PEER_IP}\"}"
```

## Step 7: Configure SNMPv3 for Secure Monitoring

For production environments, use SNMPv3 with authentication and encryption:

```text
! Configure SNMPv3 user
Router(config)# snmp-server user bgpmonitor NetOps-Group v3 \
  auth sha AuthPassw0rd! priv aes 128 PrivPassw0rd!

! Send traps with SNMPv3
Router(config)# snmp-server host 192.168.1.100 version 3 priv bgpmonitor
```

## Conclusion

SNMP traps for BGP session health provide real-time alerting without polling overhead. Enable BGP traps with `snmp-server enable traps bgp`, configure a trap receiver on your NMS, and write handler scripts that send alerts to your on-call team. Pair with periodic SNMP polls of `bgpPeerState` in tools like Zabbix or Nagios for a complete BGP monitoring solution.
