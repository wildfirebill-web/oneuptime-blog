# How to Configure LibreNMS for IPv6 Device Discovery

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: LibreNMS, IPv6, Network Monitoring, SNMP, Device Discovery, NMS

Description: A guide to configuring LibreNMS to discover and monitor IPv6-addressed network devices via SNMP, including autodiscovery and custom SNMP credentials.

LibreNMS is a full-featured network monitoring system with native IPv6 support. It can discover devices by IPv6 address, monitor them via SNMPv3, and display IPv6 interface statistics, BGP sessions, and routing tables.

## Step 1: Ensure LibreNMS Has IPv6 Connectivity

LibreNMS must be able to reach its monitored devices over IPv6:

```bash
# Test IPv6 connectivity from the LibreNMS host

ping6 2001:db8::router1

# Test SNMP over IPv6
snmpwalk -v2c -c public "[2001:db8::router1]" sysDescr
```

## Step 2: Add an IPv6 Device Manually

```bash
# Add a device by IPv6 address using the addhost script
cd /opt/librenms
./addhost.php 2001:db8::router1 v2c public

# Or add using the web UI: Devices > Add Device
# Enter the IPv6 address directly in the "Hostname or IP" field
```

## Step 3: Configure SNMP Credentials for IPv6 Devices

```bash
# Create a custom SNMP community for IPv6 devices
# In LibreNMS UI: Settings > Global Settings > SNMP > Community

# Or add via config.php
sudo nano /opt/librenms/config.php
```

```php
<?php
// config.php - LibreNMS SNMP settings for IPv6
// SNMP v2c community string(s) to try when discovering devices
$config['snmp']['community'] = ['public', 'private', 'monitoring'];

// SNMP v3 credentials for secure IPv6 device monitoring
$config['snmp']['v3'][0]['authlevel'] = 'authPriv';
$config['snmp']['v3'][0]['authname']  = 'librenms';
$config['snmp']['v3'][0]['authpass']  = 'authpassword123';
$config['snmp']['v3'][0]['authalgo']  = 'SHA';
$config['snmp']['v3'][0]['cryptopass']= 'privpassword123';
$config['snmp']['v3'][0]['cryptoalgo']= 'AES';

// Enable IPv6 support in LibreNMS
$config['ipv6'] = true;
```

## Step 4: Configure IPv6 Network Autodiscovery

```php
// config.php - Configure autodiscovery ranges including IPv6
// IPv6 network ranges for autodiscovery (uses ICMPv6 or SNMP CDP/LLDP)
$config['nets'][] = '10.0.0.0/8';           // IPv4 range
$config['nets'][] = '2001:db8::/32';        // IPv6 range

// Configure discovery method order
$config['discovery_modules']['xdp']   = true;  // CDP/LLDP discovery
$config['discovery_modules']['snmp-scan'] = true;
```

```bash
# Run discovery manually for the IPv6 network
./discovery.php -h all

# Discover a specific IPv6 host
./discovery.php -h 2001:db8::router1
```

## Step 5: Monitor IPv6 Interface Statistics

Once a device is added, LibreNMS automatically polls its interfaces including IPv6 ones. To verify:

```bash
# Check collected interface data
./poller.php -h 2001:db8::router1

# View in UI: Devices > [Device Name] > Interfaces
# IPv6 interfaces appear alongside IPv4 ones
```

## Step 6: Set Up IPv6 BGP Monitoring

LibreNMS monitors BGP sessions automatically when devices are discovered:

```bash
# Verify BGP module is enabled
grep -i bgp /opt/librenms/config.php

# Manually poll BGP data
./poller.php -h 2001:db8::router1 -m bgp

# View: Routing > BGP Sessions
# IPv6 BGP peers appear with their IPv6 neighbor addresses
```

## Step 7: Create IPv6-Specific Alerts

```php
// In LibreNMS UI: Alerts > Alert Rules > Create

// Alert when IPv6 interface goes down
// Rule: %devices.status != 1 AND %ports.ifAdminStatus = "up"

// Alert when IPv6 device is unreachable
// Rule: %devices.status = 0 AND %devices.type = "network"
```

## Step 8: API Query for IPv6 Devices

```bash
# List all devices discovered with IPv6 addresses
curl -s -H "X-Auth-Token: $LIBRENMS_API_TOKEN" \
  "https://librenms.example.com/api/v0/devices?type=network" | \
  jq '.devices[] | select(.ip | test(":")) | {hostname, ip, sysName}'
```

LibreNMS's SNMP-based discovery and polling works seamlessly with IPv6 addresses, providing the same rich monitoring capabilities for IPv6 devices as for IPv4, including interface graphs, BGP sessions, and routing table visualization.
