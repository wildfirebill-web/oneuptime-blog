# How to Fix Duplicate IPv4 Address Detection Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Duplicate IP, ARP, DHCP, Windows, Troubleshooting

Description: Learn how to detect and resolve duplicate IPv4 address errors where Windows disables the adapter after detecting another device using the same IP address.

## What Is Duplicate Address Detection?

When a device gets an IP address (via DHCP or static config), it performs ARP probing to check if that IP is already in use. If another device responds, Windows:
1. Logs Event ID 4199 in the System event log
2. May disable the adapter with the error "The IP address X.X.X.X you have entered for this network adapter is already in use"

## Step 1: Confirm Duplicate Address Detection

```cmd
REM Windows Event Viewer — check for Event ID 4199
eventvwr.msc
REM Navigate: Windows Logs → System
REM Filter: Source = "Tcpip" or Event ID = 4199

REM PowerShell check
Get-EventLog -LogName System -Source Tcpip -Newest 20 |
    Where-Object {$_.EventID -eq 4199} |
    Format-List TimeGenerated, Message

REM Check current IP status
ipconfig /all
REM If adapter shows no IP or 0.0.0.0, it was disabled due to conflict
```

## Step 2: Identify Both Conflicting Devices

```bash
# Find all devices responding to the conflicted IP
# Use arping (Linux/macOS)
sudo arping -c 5 192.168.1.50 -I eth0

# If two MACs reply, you have a conflict:
# ARPING 192.168.1.50 from 192.168.1.100 eth0
# Unicast reply from 192.168.1.50 [AA:BB:CC:DD:EE:11] 1.234ms  <- Device 1
# Unicast reply from 192.168.1.50 [11:22:33:44:55:66] 1.890ms  <- Device 2 (CONFLICT)

# Look up MAC vendor to identify device type
# AA:BB:CC = OUI prefix → check at https://macvendors.com
```

```cmd
REM Windows — check ARP cache
arp -a | findstr "192.168.1.50"

REM Scan network to find devices
nmap -sn 192.168.1.0/24
```

## Step 3: Release the Conflicting IP

```cmd
REM On the Windows device showing the error:
REM Release current (conflicted) IP
ipconfig /release

REM Wait for conflict to clear
ping 127.0.0.1 -n 5 > nul

REM Renew to get a new, non-conflicting IP
ipconfig /renew
ipconfig /all
```

## Step 4: Fix the Root Cause

**If one device has a static IP in the DHCP range:**

```bash
# Option A: Exclude the static IP from DHCP pool (ISC DHCPD)
# /etc/dhcp/dhcpd.conf — add a reservation
host conflict_device {
    hardware ethernet AA:BB:CC:DD:EE:11;
    fixed-address 192.168.1.50;
}

# Option B: Move static IP outside DHCP pool
# Change DHCP range to avoid the static IP
# range 192.168.1.100 192.168.1.200;
# And configure static device to use 192.168.1.50 (below the range)

# Restart DHCP server
sudo systemctl restart isc-dhcp-server
```

**If DHCP gave out the same IP twice (lease file corruption):**

```bash
# ISC DHCPD — clear corrupt lease file
sudo systemctl stop isc-dhcp-server
sudo cp /var/lib/dhcp/dhcpd.leases /var/lib/dhcp/dhcpd.leases.bak
sudo truncate -s 0 /var/lib/dhcp/dhcpd.leases
sudo systemctl start isc-dhcp-server
```

## Step 5: Enable DHCP Conflict Detection

```bash
# ISC DHCPD — ping-check before assigning IPs
# /etc/dhcp/dhcpd.conf
ping-check true;
ping-timeout 2;

# DHCPD will ping the IP before assigning it
# If a device responds, that IP is skipped
```

## Step 6: Monitor for Future Conflicts

```bash
#!/bin/bash
# /usr/local/bin/arp-conflict-monitor.sh
# Detect duplicate IPs in ARP table

declare -A seen_ips

while IFS= read -r line; do
    ip=$(echo "$line" | awk '{print $1}')
    mac=$(echo "$line" | awk '{print $3}')

    if [[ -n "${seen_ips[$ip]}" ]] && [[ "${seen_ips[$ip]}" != "$mac" ]]; then
        logger -p daemon.warning "IP CONFLICT: $ip has MACs ${seen_ips[$ip]} and $mac"
        echo "$(date): CONFLICT $ip: ${seen_ips[$ip]} vs $mac" >> /var/log/arp-conflicts.log
    fi
    seen_ips[$ip]=$mac
done < <(arp -n | tail -n +2)
```

```bash
# Run every minute via cron
echo "* * * * * /usr/local/bin/arp-conflict-monitor.sh" | sudo crontab -
```

## Conclusion

Duplicate IPv4 detection is confirmed via Windows Event ID 4199 or `arping` showing two MACs for one IP. Fix by releasing/renewing DHCP on the affected device, then addressing the root cause: either adding a DHCP reservation for the static-IP device or moving the static IP outside the DHCP pool. Enable `ping-check true` in ISC DHCPD to proactively avoid future conflicts, and deploy ARP monitoring to detect any that slip through.
