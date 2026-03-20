# How to Troubleshoot Slow WiFi Caused by IP Address Conflicts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WiFi, IP Conflict, Slow Network, ARP, Troubleshooting

Description: Learn how to detect and resolve IP address conflicts on WiFi networks that cause intermittent connectivity and slow performance.

## How IP Conflicts Cause Slow WiFi

When two devices share the same IP address, network traffic destined for that IP may be delivered to either device unpredictably. This causes:
- Intermittent connection drops
- Partial connectivity (some requests succeed, others fail)
- Slow response times as packets go to the wrong device
- ARP storms as both devices respond to the same ARP query

## Step 1: Detect an IP Conflict

**Windows:**
```cmd
REM Windows automatically detects conflicts and shows a notification
REM Check Event Viewer for IP conflict events:
eventvwr.msc
REM Windows Logs → System → Filter for Event ID 4199

REM Check current IP assignment
ipconfig /all

REM Scan for devices with the same IP using ARP
arp -a | findstr "192.168.1.50"
REM Multiple entries with same IP but different MAC = conflict
```

**Linux/macOS:**
```bash
# Check for conflict using arping

sudo arping -c 5 192.168.1.50 -I wlan0

# If multiple different MAC addresses respond, there's a conflict
# ARPING 192.168.1.50 from 192.168.1.100 wlan0
# Unicast reply from 192.168.1.50 [AA:BB:CC:DD:EE:FF]  <- Device 1
# Unicast reply from 192.168.1.50 [11:22:33:44:55:66]  <- Device 2 (CONFLICT!)

# Or use arp-scan to find all devices
sudo arp-scan --interface=wlan0 --localnet | sort -t'.' -k4 -n
```

## Step 2: Identify the Conflicting Devices

```bash
# Find MAC address of the conflicting device
arp -n | grep "192.168.1.50"

# Look up the manufacturer from the MAC address OUI
# First 3 octets of MAC = manufacturer
# Example: AA:BB:CC → look up at https://macvendors.com/

# Scan all devices on the network
nmap -sn 192.168.1.0/24 --open

# Show all ARP table entries
arp -a

# Find duplicate IPs
arp -a | awk '{print $2}' | sort | uniq -d
```

## Step 3: Resolve the IP Conflict

**Option A: Release and renew (for DHCP-assigned addresses)**
```bash
# Linux
sudo dhclient -r wlan0
sudo dhclient wlan0

# Windows
ipconfig /release
ipconfig /renew

# macOS
sudo ipconfig set en0 DHCP
```

**Option B: Configure DHCP to avoid conflicts**
```bash
# In the DHCP server, reserve IPs for known MAC addresses
# to prevent two devices from getting the same IP

# dnsmasq example (/etc/dnsmasq.conf):
dhcp-host=AA:BB:CC:DD:EE:FF,192.168.1.50,hostname1
dhcp-host=11:22:33:44:55:66,192.168.1.51,hostname2
```

**Option C: Find and change the static IP offender**
```bash
# If a device has a hard-coded static IP that conflicts:
# 1. Identify the device by MAC (arping or arp-scan)
# 2. Log into the device and change its static IP
# 3. Or reserve the IP in DHCP to prevent DHCP from assigning it

# In ISC DHCPD, exclude the conflicting static IP from the pool:
# range 192.168.1.100 192.168.1.199;  # Skip .50 which is statically assigned
```

## Step 4: Configure DHCP Conflict Detection

```bash
# ISC DHCPD can ping an IP before assigning it (detect static conflicts)
# /etc/dhcp/dhcpd.conf
ping-check true;
ping-timeout 2;    # Seconds to wait for ping reply

# If the IP responds to ping, DHCPD skips it and assigns the next available IP
```

## Step 5: Monitor for Future Conflicts

```bash
# Monitor ARP traffic for conflict detection
sudo tcpdump -i wlan0 arp

# Or use gratuitous ARP monitoring
# Gratuitous ARPs (sender IP = target IP) are sent when a device
# acquires a new IP and also used to detect conflicts

# Set up a cron job to scan for duplicates
cat > /usr/local/bin/detect-ip-conflicts.sh << 'EOF'
#!/bin/bash
arp -a | awk '{print $2}' | tr -d '()' | sort | uniq -d | while read ip; do
    echo "$(date): IP conflict detected for $ip" >> /var/log/ip-conflicts.log
done
EOF
crontab -e
# */5 * * * * /usr/local/bin/detect-ip-conflicts.sh
```

## Conclusion

IP conflicts on WiFi cause intermittent connectivity and slow performance that's hard to diagnose. Detect conflicts using `arping` (look for multiple MAC responses to one IP) or Windows Event ID 4199. Resolve by releasing/renewing DHCP leases on conflicting devices, reserving IPs in DHCP for known MACs, and configuring DHCPD's `ping-check` option to detect static IP conflicts before assigning. Prevent future conflicts by using DHCP reservations for all static-IP devices.
