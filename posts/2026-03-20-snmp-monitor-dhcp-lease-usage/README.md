# How to Monitor DHCP Lease Usage with SNMP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: SNMP, DHCP, Monitoring, IPv4, Lease, Network Management, OID

Description: Learn how to monitor DHCP pool utilization and lease counts using SNMP OIDs from Microsoft DHCP Server and ISC DHCP, enabling proactive capacity management.

---

DHCP pool exhaustion causes new devices to fail network connectivity. Monitoring lease utilization via SNMP enables proactive alerts before pools run out.

## SNMP OIDs for DHCP Monitoring

### Microsoft DHCP Server (Windows)

Microsoft's DHCP server exposes pool statistics via the `DHCP-MIB`:

| OID | Description |
|-----|-------------|
| `1.3.6.1.4.1.311.1.3.2.1.1.3` | Addresses in use per scope |
| `1.3.6.1.4.1.311.1.3.2.1.1.4` | Addresses available per scope |
| `1.3.6.1.4.1.311.1.3.2.1.1.2` | Subnet address (scope IP) |

```bash
# Walk all DHCP scopes on a Windows DHCP server

snmpwalk -v2c -c public 192.168.1.10 1.3.6.1.4.1.311.1.3.2.1

# Get addresses in use for all scopes
snmpwalk -v2c -c public 192.168.1.10 1.3.6.1.4.1.311.1.3.2.1.1.3

# Get available addresses
snmpwalk -v2c -c public 192.168.1.10 1.3.6.1.4.1.311.1.3.2.1.1.4
```

### ISC DHCP Server (Linux) - via Custom Script

ISC DHCP doesn't expose SNMP natively. Use a script to parse the lease file and expose it as an SNMP extend.

```bash
# /usr/local/bin/dhcp-lease-count.sh
#!/bin/bash
# Count active DHCP leases for a subnet

SUBNET="192.168.1"
LEASE_FILE="/var/lib/dhcpd/dhcpd.leases"

# Count active (not expired) leases in the subnet
ACTIVE=$(grep -B1 "binding state active" "$LEASE_FILE" | \
  grep "^lease" | grep "^lease ${SUBNET}" | wc -l)

POOL_SIZE=254  # Total IPs in the pool (x.x.x.1-254)

echo "$ACTIVE"
echo "$POOL_SIZE"
awk "BEGIN {printf \"%.1f\n\", ($ACTIVE/$POOL_SIZE)*100}"
```

```ini
# /etc/snmp/snmpd.conf - expose via SNMP extend
extend dhcp-active-leases /usr/local/bin/dhcp-lease-count.sh
```

```bash
# Query the custom extend OID
snmpwalk -v2c -c public 127.0.0.1 1.3.6.1.4.1.8072.1.3.2
```

## Prometheus Monitoring for ISC DHCP

```bash
# Install the DHCP exporter for Prometheus
go install github.com/DRuggeri/dhcp_exporter@latest

# Run it pointing at the lease file
dhcp_exporter \
  --dhcp.leases-file=/var/lib/dhcpd/dhcpd.leases \
  --web.listen-address=10.0.0.5:9667

# Prometheus scrape config
# - job_name: dhcp
#   static_configs: [{targets: ['10.0.0.5:9667']}]
```

## Setting Alerts for Pool Exhaustion

### Prometheus AlertRule

```yaml
groups:
  - name: dhcp
    rules:
      - alert: DHCPPoolNearlyExhausted
        expr: dhcp_pool_utilization_percent > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "DHCP pool {{ $labels.subnet }} is {{ $value }}% utilized"
          description: "Pool may be exhausted soon. Add more IPs or split the scope."
```

### PRTG Alert

In PRTG, create a custom SNMP sensor pointing to the Microsoft DHCP OID and set a threshold alert at 85% utilization.

## Key Takeaways

- Microsoft DHCP server exposes per-scope lease counts via OID `1.3.6.1.4.1.311.1.3.2.1.1.3`.
- ISC DHCP requires a helper script and `extend` in `snmpd.conf` to expose lease counts via SNMP.
- Alert at 80-85% utilization to provide time to add IP addresses or resize pools before exhaustion.
- For production monitoring, use a dedicated Prometheus exporter or Nagios plugin for richer DHCP metrics.
