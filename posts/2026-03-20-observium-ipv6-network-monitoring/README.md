# How to Configure Observium for IPv6 Network Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Observium, IPv6, Network Monitoring, SNMP, NMS, Visualization

Description: A guide to configuring Observium to discover and monitor network devices over IPv6, including SNMP configuration and IPv6 interface graphs.

Observium is a network monitoring platform with auto-discovery capabilities. It supports IPv6 device discovery via SNMP and can monitor IPv6 interface statistics, routing tables, and BGP sessions.

## Step 1: Enable IPv6 in Observium Configuration

```php
<?php
// /opt/observium/config.php - Enable IPv6 in Observium

// Allow Observium to discover and use IPv6 addresses
$config['ipv6'] = true;

// SNMP community strings to try when discovering devices
$config['snmp']['community'] = ['public', 'monitoring'];

// SNMP v3 for secure IPv6 monitoring
$config['snmp']['v3'][0]['authlevel']  = 'authPriv';
$config['snmp']['v3'][0]['authname']   = 'observium';
$config['snmp']['v3'][0]['authpass']   = 'authPassword123';
$config['snmp']['v3'][0]['authalgo']   = 'SHA';
$config['snmp']['v3'][0]['cryptopass'] = 'privPassword123';
$config['snmp']['v3'][0]['cryptoalgo'] = 'AES';
```

## Step 2: Add an IPv6 Device to Observium

```bash
# Add device by IPv6 address
cd /opt/observium
./addhost.php 2001:db8::router1 v2c public 161 udp

# Or add via the web UI:
# Devices > Add Device
# Enter: 2001:db8::router1 as the hostname/IP
# Select SNMP version and enter community/credentials
```

## Step 3: Configure Autodiscovery for IPv6 Networks

```php
// config.php - Set IPv6 network ranges for autodiscovery
// Networks listed here will be scanned for new SNMP-capable devices
$config['autodiscovery']['networks'][] = '10.0.0.0/8';
$config['autodiscovery']['networks'][] = '2001:db8::/32';    // IPv6 range
$config['autodiscovery']['networks'][] = 'fd00::/8';         // ULA range

// Enable xDP (CDP/LLDP) discovery which works over IPv6
$config['autodiscovery']['xdp'] = 1;
```

```bash
# Run autodiscovery for the IPv6 network
./discovery.php -h all

# Or run for a specific IPv6 range
./discovery.php -n 2001:db8::/32
```

## Step 4: Poll IPv6 Device Data

```bash
# Run the poller for a specific IPv6 device
./poller.php -h 2001:db8::router1

# Set up cron jobs for continuous polling (typical Observium setup)
# These already run from the default Observium cron; just verify:
crontab -l | grep observium
```

## Step 5: Monitor IPv6 Interfaces

Once a device is polled, navigate to the device in the Observium UI:

- **Devices → [Device Name] → Interfaces**
- IPv6 interfaces (with IPv6 addresses) appear alongside IPv4 interfaces
- Click an interface to see RX/TX graphs

Observium automatically collects:
- Interface operational status
- IPv6 address assignments from the device
- Bytes/packets in and out per IPv6 interface

## Step 6: View IPv6 Routing Table

```bash
# Poll routing table data for IPv6 device
./poller.php -h 2001:db8::router1 -m routes

# View in UI: Devices > [Device] > Routing > IPv6 Routes
```

## Step 7: Monitor IPv6 BGP Sessions

```bash
# Enable BGP polling module
# config.php
$config['poller_modules']['bgp-peers'] = 1;

# Poll BGP data
./poller.php -h 2001:db8::router1 -m bgp-peers
```

Navigate to **Routing → BGP** to see all BGP sessions including IPv6 peer addresses.

## Step 8: Set Up Email Alerts for IPv6 Device Issues

```php
// config.php - Alert configuration
$config['email']['default']     = 'admin@example.com';
$config['email']['from']        = 'observium@example.com';
$config['email']['smtp_host']   = 'smtp.example.com';
$config['email']['smtp_port']   = 587;

// Alert when a device goes down
$config['alerts']['email_down'] = 1;
```

## Verify IPv6 Monitoring

```bash
# Check SNMP connectivity to IPv6 device
snmpget -v2c -c public "[2001:db8::router1]" sysDescr.0

# Verify Observium can reach the IPv6 device
./poller.php -h 2001:db8::router1 -d

# Check Observium error log
tail -f /opt/observium/logs/observium.log | grep -i "ipv6\|error"
```

Observium's SNMP-driven monitoring model works identically for IPv6 devices as for IPv4 — simply enter the IPv6 address when adding a device and Observium handles the rest, providing full interface graphs and BGP session visibility.
