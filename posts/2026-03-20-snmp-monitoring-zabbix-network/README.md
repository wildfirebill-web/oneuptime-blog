# How to Configure SNMP Monitoring in Zabbix for Network Devices

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Zabbix, SNMP, Network Monitoring, Cisco, Dashboard, Alerting

Description: Learn how to add network devices to Zabbix using SNMP, apply built-in network device templates, and set up triggers for interface up/down alerting.

## Why Zabbix for SNMP Monitoring?

Zabbix is an enterprise-grade open-source monitoring platform with excellent SNMP support. It can poll SNMP OIDs on a schedule, receive SNMP traps, auto-discover interfaces, and trigger alerts based on thresholds.

## Step 1: Install Zabbix Server

```bash
# Install Zabbix 7.0 on Ubuntu

wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_7.0-1+ubuntu22.04_all.deb
sudo apt-get update

sudo apt-get install -y zabbix-server-mysql zabbix-frontend-php \
  zabbix-nginx-conf zabbix-sql-scripts zabbix-agent

# Configure database and start
sudo systemctl enable zabbix-server zabbix-agent nginx php8.1-fpm
sudo systemctl start zabbix-server zabbix-agent nginx php8.1-fpm
```

## Step 2: Add a Network Device Host

In the Zabbix web UI (Configuration > Hosts > Create Host):

1. **Host name:** `core-router-01`
2. **Groups:** `Network Devices`
3. **Interfaces:** Add SNMP interface
   - IP Address: `192.168.1.1`
   - Port: `161`
4. **Templates:** Apply `Network Interfaces SNMPv2`

Or use the Zabbix API to add the host programmatically:

```bash
# Add host via Zabbix API
curl -X POST http://zabbix.example.com/api_jsonrpc.php \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "host.create",
    "params": {
      "host": "core-router-01",
      "interfaces": [{
        "type": 2,
        "main": 1,
        "useip": 1,
        "ip": "192.168.1.1",
        "dns": "",
        "port": "161"
      }],
      "groups": [{"groupid": "5"}],
      "macros": [
        {"macro": "{$SNMP_COMMUNITY}", "value": "public"},
        {"macro": "{$SNMP_VERSION}", "value": "2"}
      ]
    },
    "auth": "YOUR_AUTH_TOKEN",
    "id": 1
  }'
```

## Step 3: Configure SNMP Macros

Set the SNMP community as a host macro:

In the Zabbix UI: Host > Macros tab:
- `{$SNMP_COMMUNITY}` = `Net0ps_M0n!t0r`
- `{$SNMP_VERSION}` = `2` (for SNMPv2c)

For SNMPv3, configure additional macros:
- `{$SNMP3_USER}` = `nmsuser`
- `{$SNMP3_AUTHPASSPHRASE}` = `AuthPass@2026!`
- `{$SNMP3_PRIVPASSPHRASE}` = `PrivPass@2026!`

## Step 4: Configure Interface Discovery

Zabbix built-in templates use SNMP Low-Level Discovery (LLD) to automatically find all interfaces:

The template `Network Interfaces SNMPv2` automatically:
- Discovers all interfaces via `ifDescr` (OID: 1.3.6.1.2.1.2.2.1.2)
- Monitors `ifOperStatus`, `ifInOctets`, `ifOutOctets`
- Creates triggers for interface state changes

To manually add an item for interface bandwidth:

```text
# Custom SNMP item for GigabitEthernet0/0 inbound bytes
OID: 1.3.6.1.2.1.2.2.1.10.1   (ifInOctets for interface index 1)
Type: SNMP agent
Value type: Numeric (unsigned 64-bit)
Units: bytes
Update interval: 60s
```

## Step 5: Create an Interface Down Trigger

Zabbix built-in templates include interface status triggers. To create a custom one:

```text
# Trigger expression for interface down
{core-router-01:if.status[1.3.6.1.2.1.2.2.1.8.1].last()}=2

# This fires when ifOperStatus for interface index 1 = 2 (down)
```

Use discovery-based triggers in templates to cover all interfaces automatically.

## Step 6: Receive SNMP Traps in Zabbix

Configure Zabbix to receive SNMP traps using snmptrapd:

```bash
# Install snmptrapd
sudo apt-get install -y snmptrapd

# Configure to forward to Zabbix
cat > /etc/snmp/snmptrapd.conf << 'EOF'
authCommunity execute,log,net public
traphandle default /usr/bin/zabbix_trap_receiver.pl
EOF

sudo systemctl restart snmptrapd
```

Add a trap item to the host:
- Type: `SNMP trap`
- Key: `snmptrap.fallback`

## Step 7: Verify SNMP Polling

Check that Zabbix is successfully polling the device:

1. Go to **Monitoring > Latest data**
2. Filter by the host name
3. Verify SNMP items show recent values (not "No data")

If polling fails, check:
```bash
# Test SNMP polling from Zabbix server
snmpget -v2c -c Net0ps_M0n!t0r 192.168.1.1 sysDescr.0
```

## Conclusion

Zabbix provides comprehensive SNMP monitoring with built-in templates for Cisco and other network devices. Add devices with SNMP interfaces, apply the appropriate template for automatic interface discovery, configure SNMP community macros, and let Zabbix handle polling and alerting. For production environments, configure SNMPv3 macros instead of community strings for secure monitoring.
