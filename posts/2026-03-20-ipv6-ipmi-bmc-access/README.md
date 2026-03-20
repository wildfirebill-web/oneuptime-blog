# How to Configure IPv6 for IPMI/BMC Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, IPMI, BMC, iDRAC, iLO, Remote Management, Server Hardware

Description: Configure IPv6 on IPMI and BMC (Baseboard Management Controller) interfaces for Dell iDRAC, HP iLO, Supermicro IPMI, and generic IPMI 2.0 devices for remote server management over IPv6.

---

IPMI (Intelligent Platform Management Interface) and BMC provide hardware-level remote access independent of the server's operating system. Configuring IPv6 on IPMI/BMC interfaces enables out-of-band management over IPv6 networks, including power control, console access, and hardware monitoring.

## Dell iDRAC IPv6 Configuration

```bash
# Method 1: racadm command line

# Enable IPv6
racadm set iDRAC.IPv6.Enable 1

# Configure static IPv6
racadm set iDRAC.IPv6.Address1 2001:db8:oob::201
racadm set iDRAC.IPv6.PrefixLength1 64
racadm set iDRAC.IPv6.Gateway1 2001:db8:oob::1
racadm set iDRAC.IPv6.AutoConfig Disabled

# Or enable DHCPv6
racadm set iDRAC.IPv6.AutoConfig 1
racadm set iDRAC.IPv6.Autoconfig 2  # DHCPv6 stateful

# Verify configuration
racadm get iDRAC.IPv6

# Remote racadm via existing IPv4
racadm -r 192.168.1.201 -u root -p calvin set iDRAC.IPv6.Address1 2001:db8:oob::201

# Method 2: iDRAC Web GUI
# https://idrac-ip/
# iDRAC Settings > Network > IPv6
# Enable IPv6: Enabled
# IP Address: 2001:db8:oob::201
# Prefix Length: 64
# Gateway: 2001:db8:oob::1
```

## HP iLO IPv6 Configuration

```bash
# HP iLO - Configure IPv6

# Via hponcfg XML
cat > /tmp/ilo-ipv6-config.xml << 'EOF'
<RIBCL VERSION="2.23">
  <LOGIN USER_LOGIN="admin" PASSWORD="password">
    <RIB_INFO MODE="write">
      <MOD_NETWORK_SETTINGS>
        <SPEED_AUTOSELECT VALUE="Yes"/>
        <REG_WINS_SERVER VALUE="No"/>
        <DHCP_ENABLE VALUE="No"/>
        <IP_ADDRESS VALUE="192.168.1.202"/>
        <SUBNET_MASK VALUE="255.255.255.0"/>
        <GATEWAY_IP_ADDRESS VALUE="192.168.1.1"/>
        <!-- IPv6 Settings -->
        <IPV6_ADDRESS VALUE="2001:db8:oob::202"/>
        <IPV6_PREFIX_LENGTH VALUE="64"/>
        <IPV6_DEFAULT_GATEWAY VALUE="2001:db8:oob::1"/>
        <PREFER_IPV6 VALUE="No"/>
        <DHCPV6_ENABLED VALUE="No"/>
        <IPV6_STATIC_IP_ADDRESS_1 VALUE="2001:db8:oob::202"/>
        <IPV6_STATIC_PREFIX_LENGTH_1 VALUE="64"/>
      </MOD_NETWORK_SETTINGS>
    </RIB_INFO>
  </LOGIN>
</RIBCL>
EOF

hponcfg -i < /tmp/ilo-ipv6-config.xml

# Verify via hponcfg
cat > /tmp/ilo-get-network.xml << 'EOF'
<RIBCL VERSION="2.23">
  <LOGIN USER_LOGIN="admin" PASSWORD="password">
    <RIB_INFO MODE="read">
      <GET_NETWORK_SETTINGS/>
    </RIB_INFO>
  </LOGIN>
</RIBCL>
EOF

hponcfg -i < /tmp/ilo-get-network.xml
```

## Supermicro IPMI IPv6

```bash
# Supermicro IPMI - ipmitool configuration

# Check IPv6 support
ipmitool -H 192.168.1.203 -U admin -P password lan6 print 1

# Enable IPv6 channel
ipmitool -H 192.168.1.203 -U admin -P password \
    raw 0x0c 0x01 0x01 0xc0 0x01

# Set IPv6 static address
ipmitool -H 192.168.1.203 -U admin -P password \
    lan6 set 1 ipv6static 0 address 2001:db8:oob::203

ipmitool -H 192.168.1.203 -U admin -P password \
    lan6 set 1 ipv6static 0 prefix_len 64

ipmitool -H 192.168.1.203 -U admin -P password \
    lan6 set 1 ipv6static 0 gateway 2001:db8:oob::1

# Enable the static address
ipmitool -H 192.168.1.203 -U admin -P password \
    lan6 set 1 enables ipv6_static_addr

# Verify
ipmitool -H 192.168.1.203 -U admin -P password lan6 print 1

# Test access via IPv6
ipmitool -H 2001:db8:oob::203 -U admin -P password power status
```

## Generic IPMI 2.0 IPv6 via ipmitool

```bash
# Local ipmitool (on the server itself)

# Enable IPv6 on BMC LAN channel
ipmitool lan6 set 1 enables ipv6

# Check IPv6 SLAAC (if supported)
ipmitool lan6 print 1 | grep -i slaac

# Set static IPv6 address
ipmitool lan6 set 1 ipv6static 0 address 2001:db8:oob::101
ipmitool lan6 set 1 ipv6static 0 prefix_len 64
ipmitool lan6 set 1 ipv6static 0 gateway 2001:db8:oob::1
ipmitool lan6 set 1 enables ipv6_static_addr

# Verify current IPv6 configuration
ipmitool lan6 print 1

# Check BMC SDR sensors remotely via IPv6
ipmitool -H 2001:db8:oob::101 -U admin -P password sdr list

# Remote power control
ipmitool -H 2001:db8:oob::101 -U admin -P password power status
ipmitool -H 2001:db8:oob::101 -U admin -P password power cycle

# Remote console via SOL
ipmitool -H 2001:db8:oob::101 -U admin -P password \
    -I lanplus sol activate
```

## Redfish API over IPv6

```bash
# Modern BMC access via Redfish REST API over IPv6

# Get system info
curl -sk -u admin:password \
    https://[2001:db8:oob::201]/redfish/v1/Systems/System.Embedded.1 | \
    python3 -m json.tool | grep -E "Model|MemorySize|Status|PowerState"

# Power control via Redfish IPv6
curl -sk -X POST -u admin:password \
    -H "Content-Type: application/json" \
    -d '{"ResetType": "ForceRestart"}' \
    https://[2001:db8:oob::201]/redfish/v1/Systems/System.Embedded.1/Actions/ComputerSystem.Reset

# Get sensor readings
curl -sk -u admin:password \
    https://[2001:db8:oob::201]/redfish/v1/Chassis/System.Embedded.1/Thermal | \
    python3 -m json.tool | grep -E "Name|ReadingCelsius"
```

## Mass BMC IPv6 Configuration Script

```bash
#!/bin/bash
# configure_bmc_ipv6.sh - Configure IPv6 on all BMC interfaces

# Array of servers: hostname:ipv4-bmc:ipv6-bmc
SERVERS=(
    "server-01:192.168.1.201:2001:db8:oob::201"
    "server-02:192.168.1.202:2001:db8:oob::202"
    "server-03:192.168.1.203:2001:db8:oob::203"
)

GW6="2001:db8:oob::1"
ADMIN_USER="admin"
ADMIN_PASS="password"

for server in "${SERVERS[@]}"; do
    NAME=$(echo $server | cut -d: -f1)
    IPV4=$(echo $server | cut -d: -f2)
    IPV6=$(echo $server | cut -d: -f3)

    echo "Configuring IPv6 on $NAME BMC ($IPV6)..."

    ipmitool -H $IPV4 -U $ADMIN_USER -P $ADMIN_PASS \
        lan6 set 1 ipv6static 0 address $IPV6 && \
    ipmitool -H $IPV4 -U $ADMIN_USER -P $ADMIN_PASS \
        lan6 set 1 ipv6static 0 prefix_len 64 && \
    ipmitool -H $IPV4 -U $ADMIN_USER -P $ADMIN_PASS \
        lan6 set 1 ipv6static 0 gateway $GW6 && \
    ipmitool -H $IPV4 -U $ADMIN_USER -P $ADMIN_PASS \
        lan6 set 1 enables ipv6_static_addr && \
    echo "$NAME: IPv6 configured successfully" || \
    echo "$NAME: FAILED to configure IPv6"
done

echo "Verifying IPv6 BMC connectivity..."
for server in "${SERVERS[@]}"; do
    NAME=$(echo $server | cut -d: -f1)
    IPV6=$(echo $server | cut -d: -f3)
    ping6 -c 1 -W 2 $IPV6 > /dev/null 2>&1 && \
        echo "$NAME ($IPV6): REACHABLE" || \
        echo "$NAME ($IPV6): UNREACHABLE"
done
```

IPMI/BMC IPv6 configuration varies by vendor but follows the same pattern: enable IPv6 on the BMC LAN channel, assign a static IPv6 address from the OOB management prefix, configure the IPv6 gateway, and verify with remote ipmitool commands or Redfish API calls using bracket notation for the IPv6 address in the URL.
