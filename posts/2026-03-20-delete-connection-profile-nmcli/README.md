# How to Delete a Connection Profile with nmcli

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: nmcli, NetworkManager, Connection Profile, Linux, RHEL, Cleanup, Networking

Description: Learn how to delete NetworkManager connection profiles using nmcli, including safe deletion procedures, cleanup of associated interface configurations, and batch deletion.

---

NetworkManager stores connection profiles that persist across reboots. Deleting stale or unwanted profiles prevents them from reconnecting automatically and keeps the configuration clean.

## Listing Connection Profiles

```bash
# List all connection profiles
nmcli connection show

# Output:
# NAME           UUID                                  TYPE      DEVICE
# eth0-static    abc123-...                            ethernet  eth0
# bond0          def456-...                            bond      bond0
# vpn-work       ghi789-...                            vpn       --

# List active connections only
nmcli connection show --active
```

## Deleting a Connection Profile

```bash
# Delete by name
nmcli connection delete eth0-static

# Delete by UUID
nmcli connection delete abc123-def456-ghi7-jkl8-mnopqrstuvwx

# Verify deletion
nmcli connection show | grep eth0-static
# (empty output = deleted)
```

## Safely Deleting an Active Connection

```bash
# Bringing down before deleting (if currently active)
nmcli connection down eth0-static

# Then delete
nmcli connection delete eth0-static

# Alternatively, delete brings it down automatically
nmcli connection delete eth0-static  # Works even if active
```

## Batch Deletion

```bash
# Delete all connection profiles for a specific type
nmcli connection show | grep ethernet | awk '{print $1}' | while read name; do
  echo "Deleting: $name"
  nmcli connection delete "$name"
done

# Delete all VPN connections
nmcli connection show | grep vpn | awk '{print $1}' | \
  xargs -I{} nmcli connection delete {}

# Delete profiles with no associated device
nmcli connection show | grep " -- " | awk '{print $1}' | \
  while read name; do
    nmcli connection delete "$name"
  done
```

## Deleting and Recreating Clean

```bash
# Export current connection settings before deletion
nmcli connection show <name> > /tmp/connection-backup.txt

# Delete
nmcli connection delete <name>

# If needed, recreate from scratch:
nmcli connection add type ethernet ifname eth0 con-name eth0-new \
  ipv4.method manual ipv4.addresses 10.0.0.5/24 ipv4.gateway 10.0.0.1
```

## Connection File Locations

NetworkManager stores profiles as files:

```bash
# RHEL/CentOS (keyfile format, NM >= 1.20)
ls /etc/NetworkManager/system-connections/

# Legacy ifcfg format (older RHEL)
ls /etc/sysconfig/network-scripts/ifcfg-*

# Manual deletion of profile file (not recommended over nmcli)
rm /etc/NetworkManager/system-connections/eth0-static.nmconnection
nmcli connection reload  # Tell NM to reread connections
```

## Key Takeaways

- `nmcli connection delete <name>` removes the profile and disconnects the interface if active.
- Use `nmcli connection show` to list all profiles and identify stale ones (those with no device in the DEVICE column).
- Connection files are stored in `/etc/NetworkManager/system-connections/`; avoid deleting files directly — use nmcli.
- After deleting all connections for an interface, `nmcli device disconnect <iface>` ensures the interface is fully brought down.
