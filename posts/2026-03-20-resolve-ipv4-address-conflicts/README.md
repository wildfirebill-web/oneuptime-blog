# How to Resolve IPv4 Address Conflicts on a Network

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv4, Address Conflict, ARP, DHCP, Troubleshooting

Description: Learn how to detect, identify, and resolve IPv4 address conflicts on a network where two devices share the same IP address, causing connectivity issues.

## How IP Conflicts Occur

An IP conflict happens when two devices are assigned the same IPv4 address:
- DHCP assigns an IP that was previously given to a statically-configured device
- Manual configuration duplicates an existing address
- DHCP lease file corruption causes a duplicate lease
- VPN client assigns an IP matching the local network

## Step 1: Detect an IP Conflict

**Windows (automatic detection):**
```cmd
REM Windows displays a notification and logs Event ID 4199
eventvwr.msc → Windows Logs → System → Filter for "Tcpip" or Event ID 4199

REM Check current IP assignment
ipconfig /all
REM Note the IP and MAC address shown
```

**Linux detection:**
```bash
# Use arping to find multiple hosts with the same IP
sudo arping -c 5 192.168.1.50 -I eth0

# If two different MAC addresses respond, there's a conflict:
# Unicast reply from 192.168.1.50 [AA:BB:CC:DD:EE:FF]  <- MAC 1
# Unicast reply from 192.168.1.50 [11:22:33:44:55:66]  <- MAC 2 (CONFLICT)
```

## Step 2: Identify Both Conflicting Devices

```bash
# Get MAC addresses of both conflicting devices
arp -n | grep "192.168.1.50"

# Look up manufacturer from MAC OUI prefix
# AA:BB:CC = OUI → look up at https://macvendors.com/
# This helps identify what device type it is

# Scan network to find all device information
nmap -sn 192.168.1.0/24 --open
```

## Step 3: Resolve the Conflict

**Option A: Release and Renew DHCP**
```bash
# Linux
sudo dhclient -r eth0
sudo dhclient eth0

# Windows
ipconfig /release
ipconfig /renew
```

**Option B: Configure DHCP to Avoid the Conflict**
```bash
# ISC DHCPD: Add DHCP reservation for the static-IP device
# to prevent DHCP from assigning that IP to another device

host static_device {
    hardware ethernet AA:BB:CC:DD:EE:FF;
    fixed-address 192.168.1.50;
}

# Or exclude the static IP from the DHCP pool
# Change: range 192.168.1.100 192.168.1.200;
# The static IP 192.168.1.50 is below the range, so safe
```

**Option C: Change the Static IP Device**
```bash
# Log into the device with the static IP and change it
# to an IP outside the DHCP pool range
```

## Step 4: Prevent Future Conflicts

```bash
# ISC DHCPD: Enable ping-check to detect conflicts before assigning
# /etc/dhcp/dhcpd.conf
ping-check true;
ping-timeout 2;    # Wait 2 seconds for ping reply

# If the IP responds to ping, DHCPD skips it and tries the next IP
# This prevents assigning an IP to a new device if a static-IP
# device is already using it
```

## Step 5: Monitor for Conflicts

```bash
# Cron job to detect conflicts
cat > /usr/local/bin/check-conflicts.sh << 'EOF'
#!/bin/bash
# Scan for duplicate MACs in ARP table
arp -n | awk '{print $1}' | sort | uniq -d | while read ip; do
    echo "$(date): Potential conflict: $ip" >> /var/log/arp-conflicts.log
done
EOF
chmod +x /usr/local/bin/check-conflicts.sh

# Run every 5 minutes
echo "*/5 * * * * /usr/local/bin/check-conflicts.sh" | crontab -
```

## Conclusion

IP conflicts are detected via Windows Event ID 4199 or `arping` on Linux (two different MACs responding to one IP). Resolve by releasing/renewing DHCP on conflicting devices, adding DHCP reservations for statically-configured devices, or changing one device's IP. Prevent future conflicts with DHCPD's `ping-check true` option and by maintaining DHCP reservations for all devices with static IPs, keeping them outside the DHCP pool range.
