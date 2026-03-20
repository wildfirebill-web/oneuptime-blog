# How to Configure SAP Systems for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SAP, IPv6, Enterprise, ERP, ABAP, NetWeaver, Business Suite

Description: Configure SAP NetWeaver and SAP S/4HANA systems to operate in IPv6 and dual-stack environments, covering instance profiles, system parameters, and network configuration.

---

SAP systems built on NetWeaver support IPv6 through profile parameters. Enabling IPv6 requires configuration of the ABAP Application Server, message server, and associated components to listen on IPv6 interfaces.

## SAP IPv6 Support Overview

```
SAP IPv6 support requirements:
- SAP Basis release 7.0 or higher for basic IPv6 support
- SAP NetWeaver 7.4+ for full dual-stack IPv6
- SAP S/4HANA: full IPv6 support
- Underlying OS must have IPv6 enabled

Architecture considerations:
- SAP Application Server (AS ABAP)
- SAP Message Server
- SAP Dispatcher
- SAP Web Dispatcher
- SAP HANA Database (separate IPv6 configuration)
```

## SAP ABAP Instance Profile for IPv6

```
# /usr/sap/<SID>/SYS/profile/<SID>_DVEBMGS<NN>_<hostname>

# Enable IPv6 support
icm/server_port_0 = PROT=HTTP,PORT=8000,TIMEOUT=120

# For explicit IPv6 binding (dual-stack):
# icm/bind_addr = <IPv6-address>
# Or for all interfaces (default):
# icm/bind_addr = 0.0.0.0 (IPv4)
# icm/bind_addr = :: (all IPv6)

# Message server hostname (use FQDN with AAAA record)
rdisp/mshost = sap-ms.example.com

# ABAP RFC destination configuration supports IPv6
# via FQDN with AAAA record in SM59

# Enqueue server
enque/server_port = 3201
# Use FQDN for IPv6-capable enqueue server
```

## SAP Web Dispatcher IPv6 Configuration

```
# /usr/sap/<SID>/WebDisp/profile/<SID>_WebDisp_<host>

# Web Dispatcher listener on IPv6
icm/server_port_0 = PROT=HTTP,PORT=80,TIMEOUT=120

# For explicit IPv6 binding:
# icm/bind_addr = ::

# Backend connection to AS ABAP over IPv6
# Use FQDN: icm/HTTP/list_protos or wdisp/system_<N> with IPv6-capable hostname

# Example backend with IPv6:
wdisp/system_0 = SID=<SID>, MSHOST=<IPv6-FQDN>, MSSYSPORT=8100
```

## SAP HANA IPv6 Configuration

```bash
# SAP HANA IPv6 configuration via hdbsql

# Check current network config
hdbsql -n localhost:30013 -u SYSTEM -p password \
  "SELECT * FROM SYS.M_CONFIGURATION_PARAMETER_VALUES WHERE KEY = 'listeninterface'"

# Enable IPv6 in global.ini
# [communication]
# listeninterface = .global   # Listen on all interfaces

# Or edit /hana/shared/<SID>/global/hdb/custom/config/global.ini
# [communication]
# listeninterface = .global

# Restart HANA after changes
sudo HDB restart
```

## SAP Router for IPv6

```
# saprouttab (SAP Router table) with IPv6

# Allow connection from IPv6 network
P  2001:db8:client::/48  *  3299

# Forward DIAG to SAP system via IPv6
T  2001:db8::saprouter  2001:db8::sap-system  32<NN>

# Start SAP Router with IPv6
saprouter -r -n <NN> -R /usr/sap/saprouter/saprouttab

# Check SAP Router status
saprouter -l
```

## SAP RFC Destinations over IPv6

```
Configure RFC Destination (SM59) for IPv6:

1. Go to SM59 in SAP GUI
2. Create RFC Destination type "TCP/IP"
3. Technical Settings:
   - Activation Type: Start on Explicit Host
   - Target Host: ipv6server.example.com
     (Use FQDN with AAAA record, not raw IPv6 address)
   - System Number: 00
4. Test Connection

Note: SAP GUI and RFC typically work best with
fully qualified hostnames that resolve to IPv6 addresses
rather than raw IPv6 address notation
```

## Verifying SAP IPv6 Configuration

```bash
# Check SAP processes are listening on IPv6
ss -6 -tlnp | grep -E "32|80|8000|36"

# Test connectivity to SAP system over IPv6
telnet [2001:db8::sap-system] 3200  # DIAG port
curl -6 http://[2001:db8::sap-web]/sap/bc/ping

# Check OS-level hosts file for SAP hostnames
grep "sap" /etc/hosts

# Verify FQDN resolves to IPv6
dig AAAA sap.example.com +short
```

SAP systems support IPv6 through instance profile parameters and the ICM (Internet Communication Manager) layer, with the recommended approach using FQDNs with AAAA records rather than raw IPv6 addresses in SAP configuration, ensuring compatibility across all SAP components and client tools.
