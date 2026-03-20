# How to Monitor DHCP Server Logs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, Monitoring, Logging, Linux, Sysadmin, Network Diagnostics

Description: DHCP server logs record every lease assignment, renewal, release, and error, providing an audit trail for network troubleshooting, security analysis, and capacity planning.

## ISC dhcpd Logging

By default, ISC dhcpd logs to syslog (typically `/var/log/syslog` or via journald):

```bash
# Real-time log monitoring

journalctl -u isc-dhcp-server -f

# Recent log entries
journalctl -u isc-dhcp-server -n 100

# Filter for specific event types
journalctl -u isc-dhcp-server | grep -E "DHCPDISCOVER|DHCPOFFER|DHCPREQUEST|DHCPACK|DHCPNAK"

# On systems using syslog
tail -f /var/log/syslog | grep dhcpd
grep dhcpd /var/log/syslog | tail -50
```

## Enabling Detailed Logging in dhcpd

```text
# /etc/dhcp/dhcpd.conf
log-facility local7;
```

```bash
echo "local7.* /var/log/dhcpd.log" | sudo tee /etc/rsyslog.d/dhcpd.conf
sudo systemctl restart rsyslog
```

## Parsing Log Entries

Typical log entries:
```text
DHCPDISCOVER from aa:bb:cc:dd:ee:ff via eth0
DHCPOFFER on 192.168.1.105 to aa:bb:cc:dd:ee:ff via eth0
DHCPREQUEST for 192.168.1.105 (10.0.0.53) from aa:bb:cc:dd:ee:ff via eth0
DHCPACK on 192.168.1.105 to aa:bb:cc:dd:ee:ff (hostname) via eth0
```

## Python: Parsing DHCP Logs

```python
import re
from collections import defaultdict

def parse_dhcp_log(logfile: str) -> dict:
    """Parse ISC dhcpd log entries and summarize by event type."""
    pattern = re.compile(
        r'(DHCP\w+) (?:on|for|from) (\d+\.\d+\.\d+\.\d+).*?'
        r'([0-9a-f]{2}(?::[0-9a-f]{2}){5})',
        re.IGNORECASE
    )
    events = defaultdict(list)
    with open(logfile) as f:
        for line in f:
            m = pattern.search(line)
            if m:
                event, ip, mac = m.groups()
                events[event].append((ip, mac))
    return events

events = parse_dhcp_log("/var/log/dhcpd.log")
for event_type, entries in sorted(events.items()):
    print(f"{event_type:15s}: {len(entries)} events")
```

## Monitoring Lease Pool Utilization

```bash
# Count currently active leases
grep -c "binding state active" /var/lib/dhcp/dhcpd.leases

# Show pool statistics (Windows DHCP Server PowerShell)
# Get-DhcpServerv4ScopeStatistics -ScopeId 192.168.1.0
```

## Windows Server DHCP Audit Logs

Windows DHCP audit log location:
```text
C:\Windows\System32\dhcp\DhcpSrvLog-Mon.log
```

Parse with PowerShell:
```powershell
Get-Content "C:\Windows\System32\dhcp\DhcpSrvLog-Mon.log" | Select-Object -Last 50
```

## Key Takeaways

- `journalctl -u isc-dhcp-server -f` provides real-time DHCP event monitoring on Linux.
- Grep for DHCPNAK and DHCPDECLINE messages to identify lease problems quickly.
- Windows DHCP audit logs are in `C:\Windows\System32\dhcp\` and rotated daily.
- Log parsing scripts help correlate IPs to MAC addresses for security investigations.
