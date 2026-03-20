# How to Set Up DHCP on pfSense

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DHCP, pfSense, Networking, Firewall, Sysadmin

Description: pfSense's built-in DHCP server (based on ISC dhcpd) provides a web UI for configuring scopes, reservations, and options per interface, making it easy to manage DHCP alongside firewall and routing...

## Enabling DHCP via the Web UI

1. Log into pfSense WebGUI (default: https://192.168.1.1).
2. Go to **Services → DHCP Server**.
3. Select the interface (e.g., **LAN**).
4. Check **Enable DHCP server on LAN interface**.
5. Configure:
   - **Range**: Start and end IP (e.g., .100 to .200)
   - **DNS Servers**: Leave blank to use pfSense's DNS resolver
   - **Gateway**: Leave blank for pfSense's interface IP
   - **Domain Name**: (optional)
   - **Default Lease Time**: 86400 (24 hours)
   - **Maximum Lease Time**: 604800 (7 days)
6. Click **Save**.

## pfSense CLI: DHCP Configuration File

pfSense stores dhcpd.conf at `/var/dhcpd/etc/dhcpd.conf`. You can view it via SSH:

```bash
# Connect via SSH (if enabled in pfSense: System > Advanced > Admin Access)

ssh admin@192.168.1.1

# View the generated config
cat /var/dhcpd/etc/dhcpd.conf
```

## Adding a Static DHCP Mapping

Via Web UI:
1. **Services → DHCP Server → [Interface]**
2. Scroll to **DHCP Static Mappings**.
3. Click **Add**.
4. Enter MAC address and IP.
5. Save.

Via CLI script:
```php
# pfSense uses PHP config functions
# Add via Web API or GUI - direct config file editing not recommended
```

## Additional DHCP Options in pfSense

In the DHCP Server settings, scroll to **Additional BOOTP/DHCP Options**:

```text
Number: 150
Type: IP Address
Value: 10.0.0.100
```

This adds custom options like option 150 (Cisco TFTP) that aren't in the standard GUI.

## Verifying Leases

1. Go to **Status → DHCP Leases**.
2. View all active and expired leases with MAC addresses.
3. Click **Add Static Mapping** next to any lease to convert a dynamic assignment to a reservation.

Or via CLI:
```bash
# View active leases
cat /var/dhcpd/var/db/dhcpd.leases | grep -A5 "binding state active"
```

## Troubleshooting DHCP on pfSense

```bash
# Check DHCP daemon status
/usr/local/sbin/dhcpd -t  # Test config
ps aux | grep dhcpd       # Check if running

# View DHCP logs (System > System Logs > DHCP)
# Or CLI:
clog /var/log/dhcpd.log | tail -50
```

## Key Takeaways

- pfSense's DHCP server is configured per-interface with an intuitive Web UI.
- Static mappings (reservations) can be added from the Leases page by clicking existing leases.
- Use "Additional DHCP Options" for non-standard options like option 150 for Cisco phones.
- View live leases at Status → DHCP Leases to see current assignments.
