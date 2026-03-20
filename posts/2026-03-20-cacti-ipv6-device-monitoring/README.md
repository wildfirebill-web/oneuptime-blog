# How to Configure Cacti for IPv6 Device Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cacti, IPv6, SNMP, Network Monitoring, Graphing, RRDtool

Description: Configure Cacti network monitoring tool to poll and graph performance data from IPv6-addressed devices using SNMP over IPv6 transport.

---

Cacti is a complete network graphing solution using RRDtool. Configuring Cacti to monitor IPv6 devices requires adding IPv6-addressed hosts and ensuring the PHP SNMP extension and Net-SNMP libraries support IPv6 polling.

## Prerequisites for IPv6 SNMP Polling

```bash
# Verify Net-SNMP supports IPv6
snmpget --version
# Should show "NET-SNMP version 5.x" with IPv6 support

# Test SNMP over IPv6 from Cacti server
snmpget -v2c -c public udp6:[2001:db8::device]:161 sysDescr.0

# Install PHP SNMP extension
sudo apt install php-snmp -y

# Verify PHP SNMP works with IPv6
php -r "echo snmpget('udp6:[2001:db8::device]', 'public', 'sysDescr.0');"
```

## Installing Cacti

```bash
# Ubuntu/Debian
sudo apt install cacti cacti-spine -y

# Or manual install
sudo apt install apache2 php php-snmp php-mysql \
  mysql-server rrdtool -y

# Download Cacti
wget https://www.cacti.net/downloads/cacti-1.2.25.tar.gz
tar xf cacti-1.2.25.tar.gz
sudo mv cacti-1.2.25 /var/www/html/cacti

# Setup database
mysql -u root -e "CREATE DATABASE cacti;"
mysql -u root cacti < /var/www/html/cacti/cacti.sql

# Configure cacti
sudo nano /var/www/html/cacti/include/config.php
```

## Adding IPv6 Devices to Cacti

```
Via Cacti Web Interface:
1. Go to Management > Devices > Add
2. In "Hostname" field, enter IPv6 address in brackets: [2001:db8::device]
   Or use hostname with AAAA record: device.example.com
3. Set SNMP Version (v2c or v3)
4. Enter SNMP Community: public
5. Select "Associated Template": Linux Host or applicable
6. Click Create

Important: Cacti stores the hostname as-is for SNMP polling
PHP SNMP recognizes [IPv6] bracket notation for udp6 transport
```

## Cacti Configuration for IPv6 SNMP

```php
// /var/www/html/cacti/include/config.php
// Ensure snmp libraries are configured

// If using custom SNMP path
$config['snmp_timeout'] = 500;
$config['snmp_retries'] = 3;

// PHP SNMP options
// ini_set('snmp.oid_numeric_print', 1);
```

```bash
# Test Cacti can reach device over IPv6
# From Cacti server:
php -r "
\$result = snmp2_get('[2001:db8::device]', 'public', 'sysDescr.0');
var_dump(\$result);
"
```

## Spine Poller for IPv6

```bash
# /etc/cacti/spine.conf - Spine poller configuration

DB_Host     127.0.0.1
DB_Database cacti
DB_User     cactiuser
DB_Pass     password
DB_Port     3306

# Spine uses Net-SNMP which supports IPv6
# No special IPv6 config needed - uses device hostname as stored
# Ensure Net-SNMP is compiled with IPv6 support:
snmpget --version 2>&1 | grep "IPv6"
```

## Creating IPv6-Specific Graphs

```
In Cacti Web Interface:

1. Create Data Input Method:
   - Input Type: SNMP
   - OID: ipSystemStatsHCInReceives.ipv6

2. Create Data Template for IPv6 traffic:
   - Data Input: Your IPv6 SNMP input method
   - Internal Data Source Name: ipv6_in_pkts

3. Create Graph Template:
   - Graph Template Items pointing to IPv6 data templates

4. Associate with device
```

## Troubleshooting IPv6 Polling in Cacti

```bash
# Test SNMP directly from Cacti server
snmpget -v2c -c public \
  -t 2 \
  udp6:[2001:db8::device]:161 \
  1.3.6.1.2.1.1.1.0

# Check Cacti log for SNMP errors
sudo tail -f /var/www/html/cacti/log/cacti.log | grep -i "error\|snmp"

# Test PHP SNMP
php << 'EOF'
$host = "[2001:db8::device]";
$community = "public";
$oid = "1.3.6.1.2.1.1.1.0";
$result = snmp2_get($host, $community, $oid);
echo $result . "\n";
EOF

# Check firewall allows SNMP from Cacti server
sudo ip6tables -A INPUT -p udp -s 2001:db8::cacti --dport 161 -j ACCEPT
```

Cacti monitors IPv6-addressed devices by accepting IPv6 addresses in bracket notation in the hostname field, delegating actual IPv6 SNMP transport to the underlying Net-SNMP and PHP SNMP libraries which handle the `udp6` transport automatically.
