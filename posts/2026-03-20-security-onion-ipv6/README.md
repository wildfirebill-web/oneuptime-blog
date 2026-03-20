# How to Configure SecurityOnion for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SecurityOnion, IPv6, NSM, IDS, Network Security Monitoring, SOC

Description: Configure Security Onion network security monitoring platform to capture, analyze, and alert on IPv6 network traffic across your enterprise network.

---

Security Onion is a free and open platform for threat hunting, enterprise security monitoring, and log management. It integrates Suricata, Zeek, and Elastic Stack to provide comprehensive IPv6 network visibility.

## Installing Security Onion

```bash
# Security Onion is typically installed from ISO

# Download from: https://securityonionsolutions.com/software

# After installation, setup wizard
sudo so-setup

# Choose: Standalone, Manager, Sensor, etc.
# Configure management interface
# Configure monitoring interface (for sniffing)
```

## Configuring Security Onion for IPv6 Monitoring

```bash
# Check current network configuration
ip -6 addr show

# Configure monitoring interface (promiscuous mode)
sudo ip link set eth1 promisc on

# Security Onion stores config in /opt/so/conf/
ls /opt/so/conf/

# Suricata configuration
sudo nano /opt/so/conf/suricata/suricata.yaml
```

```yaml
# /opt/so/conf/suricata/suricata.yaml
vars:
  address-groups:
    HOME_NET: "[192.168.0.0/16,10.0.0.0/8,172.16.0.0/12,2001:db8::/32,fd00::/8]"
    EXTERNAL_NET: "!$HOME_NET"
```

## Adding IPv6 Detection Rules in Security Onion

```bash
# Security Onion rules are managed via so-rule
# List current rules
sudo so-rule --list | head -20

# Add custom IPv6 rule
cat >> /opt/so/rules/local.rules << 'EOF'
# IPv6 ICMPv6 RA flood detection
alert icmp6 $EXTERNAL_NET any -> $HOME_NET any \
  (msg:"SO IPv6 RA Flood Detected"; itype:134; \
   threshold:type threshold,track by_src,count 10,seconds 5; \
   sid:9900001; rev:1;)

# IPv6 address scan detection
alert icmp6 any any -> $HOME_NET any \
  (msg:"SO IPv6 Address Scan NS Flood"; itype:135; \
   threshold:type threshold,track by_src,count 50,seconds 10; \
   sid:9900002; rev:1;)
EOF

# Apply rules
sudo so-rule --reload
```

## Zeek IPv6 Analysis in Security Onion

```bash
# View Zeek conn.log for IPv6 connections
sudo so-zeek-logs conn | \
  grep ":" | \
  awk '{print $3, $5, $7}' | head -20

# IPv6 DNS queries (AAAA records)
sudo so-zeek-logs dns | \
  awk '$14 == "AAAA"' | head -20

# IPv6 HTTP traffic
sudo so-zeek-logs http | \
  awk '$3 ~ /:/' | head -20
```

## Using Security Onion Console for IPv6 Investigation

```bash
# Access Security Onion Console (SOC)
# Navigate to https://<manager-ip>

# Search for IPv6 events in Kibana/OpenSearch
# Query: data.srcip:2001\:* OR destination.ip:2001\:*

# Use Hunt for IPv6 threat hunting
# Query: network.type: "ipv6"
```

## IPv6 PCAP Capture in Security Onion

```bash
# Capture IPv6 traffic
sudo so-capture -i eth1 -f "ip6" -w /tmp/ipv6-capture.pcap

# Replay PCAP for rule testing
sudo so-import-pcap /tmp/ipv6-capture.pcap

# Download PCAP from Security Onion
# Use the "PCAP" button in alerts interface
```

## Network Sensor Deployment for IPv6

```bash
# For distributed deployments, configure sensors to monitor IPv6 VLANs
# /opt/so/conf/sensor/sensor.yaml
# monitor_interfaces:
#   - eth1  # Trunk port with all VLANs including IPv6
#   - eth2  # IPv6-only segment mirror

# Verify sensor is capturing IPv6
sudo tcpdump -i eth1 -nn ip6 -c 5
```

## Alerting on IPv6 Threats

```bash
# Configure email alerts for IPv6 events
# /opt/so/conf/so-alert.yaml

# Test alert pipeline
sudo so-alert --test

# View recent alerts
sudo so-status
sudo cat /opt/so/log/suricata/*.log | grep "ipv6\|2001:"
```

Security Onion's integrated Suricata, Zeek, and Elastic Stack provide a complete platform for IPv6 network security monitoring, with the `HOME_NET` variable in Suricata's configuration being the primary customization point for defining your IPv6 address space.
