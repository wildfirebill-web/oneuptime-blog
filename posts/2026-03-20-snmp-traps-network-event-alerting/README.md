# How to Set Up SNMP Traps for Network Event Alerting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SNMP, Traps, Alerting, Cisco IOS, Network Monitoring, Snmptrapd

Description: Learn how to configure SNMP traps on network devices and set up a Linux trap receiver to alert on network events like interface failures, BGP drops, and authentication errors.

## What Are SNMP Traps?

SNMP traps are unsolicited notifications sent by a network device to a management station when a specific event occurs (link down, authentication failure, temperature threshold, etc.). Unlike polling (which checks at intervals), traps provide near-real-time alerting with minimal overhead.

## Common Trap Categories

| Trap Type | Description |
|---|---|
| linkDown/linkUp | Interface status change |
| coldStart/warmStart | Router rebooted |
| authenticationFailure | Wrong SNMP community |
| bgpBackwardTransition | BGP session dropped |
| ospfNbrStateChange | OSPF neighbor state change |
| envMonTemperature | Temperature threshold |

## Step 1: Configure Traps on the Cisco Device

```text
! Define where to send traps
Router(config)# snmp-server host 192.168.1.100 version 2c public

! Enable the specific traps you need
Router(config)# snmp-server enable traps snmp authentication linkdown linkup
Router(config)# snmp-server enable traps bgp
Router(config)# snmp-server enable traps ospf state-change
Router(config)# snmp-server enable traps config                ! Config changes
Router(config)# snmp-server enable traps syslog               ! Syslog-level events
Router(config)# snmp-server enable traps envmon temperature   ! Temperature alerts

! Set source interface for traps
Router(config)# snmp-server trap-source Loopback0

! Set retry parameters
Router(config)# snmp-server trap-timeout 30    ! Retry every 30 seconds
```

## Step 2: Install and Configure snmptrapd on Linux

```bash
# Install net-snmp tools

sudo apt-get install -y snmp snmptrapd

# Configure snmptrapd to accept traps
cat > /etc/snmp/snmptrapd.conf << 'EOF'
# Accept SNMPv2c traps with community "public"
authCommunity log,execute,net public

# Log all traps
traphandle default /usr/local/bin/handle_snmp_trap.sh

# Listen on UDP 162 (default SNMP trap port)
snmpTrapdAddr udp:162
EOF

# Start snmptrapd
sudo systemctl enable snmptrapd
sudo systemctl start snmptrapd
```

## Step 3: Write a Trap Handler Script

```bash
#!/bin/bash
# /usr/local/bin/handle_snmp_trap.sh
# Called by snmptrapd for each received trap

# Read trap data from stdin
TRAP_DATA=$(cat)
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
HOSTNAME=$(echo "$TRAP_DATA" | head -1)
TRAP_OID=$(echo "$TRAP_DATA" | grep snmpTrapOID | awk '{print $NF}')

# Log all traps
echo "${TIMESTAMP} | ${HOSTNAME} | ${TRAP_OID}" >> /var/log/snmp_traps.log

# Alert on interface down
if echo "$TRAP_OID" | grep -q "linkDown"; then
    INTERFACE=$(echo "$TRAP_DATA" | grep ifDescr | awk '{print $NF}')
    echo "${TIMESTAMP} ALERT: ${HOSTNAME} interface ${INTERFACE} went DOWN" >> /var/log/alerts.log
    # Send webhook alert
    curl -s -X POST "https://hooks.example.com/alerts" \
         -H "Content-Type: application/json" \
         -d "{\"text\": \"Interface DOWN: ${HOSTNAME} - ${INTERFACE}\"}"
fi

# Alert on BGP session drop
if echo "$TRAP_OID" | grep -q "bgpBackwardTransition"; then
    echo "${TIMESTAMP} CRITICAL: BGP session dropped on ${HOSTNAME}" >> /var/log/alerts.log
fi
```

```bash
chmod +x /usr/local/bin/handle_snmp_trap.sh
```

## Step 4: Test Trap Delivery

Generate a test trap from the NMS server:

```bash
# Send a test trap to verify the receiver is working
snmptrap -v2c -c public 192.168.1.100 '' \
  coldStart.0 \
  sysDescr.0 s "Test Trap from NMS"

# Monitor incoming traps
tail -f /var/log/snmp_traps.log
```

Simulate a linkDown trap by shutting down an interface on the router:

```text
Router(config)# interface GigabitEthernet0/1
Router(config-if)# shutdown
```

You should see a linkDown trap in the log within seconds.

## Step 5: Integrate with Monitoring Tools

Many monitoring platforms (Zabbix, Nagios, Grafana, etc.) have built-in SNMP trap receivers. For Zabbix:

```xml
<!-- zabbix_trappers.conf snippet -->
<!-- Define a trap item linked to a host -->
<!-- Type: SNMP trap, Key: snmptrap.fallback -->
```

## Step 6: Verify Traps Are Arriving

```bash
# Check snmptrapd is listening
ss -lunp | grep 162

# View received traps in snmptrapd log
journalctl -u snmptrapd -f

# Check the trap log file
tail -50 /var/log/snmp_traps.log
```

## Conclusion

SNMP traps provide real-time alerting for critical network events. Configure trap destinations and enable specific trap categories on your network devices, then set up `snmptrapd` on a Linux NMS to receive and process them. Write handler scripts that parse trap data and forward alerts to your incident management system or monitoring platform for actionable notifications.
