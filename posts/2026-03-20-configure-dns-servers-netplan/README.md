# How to Configure DNS Servers with Netplan - Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Networking, Linux, Netplan, DNS, Ubuntu, Network Configuration

Description: Learn how to configure custom DNS servers and search domains on Ubuntu and Debian systems using Netplan.

---

Netplan is the default network configuration tool on Ubuntu 20.04+. DNS settings in Netplan are applied through `systemd-resolved` and persist across reboots.

---

## Locate the Netplan Configuration

```bash
ls /etc/netplan/
# 00-installer-config.yaml  or  01-netcfg.yaml

```

---

## Configure Static DNS

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false          # Don't use DHCP-assigned DNS
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
          - 2606:4700:4700::1111  # IPv6 DNS
        search:
          - example.com
          - internal.example.com
```

---

## Static IP with DNS

```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.1.10/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 192.168.1.53   # Internal DNS
          - 8.8.8.8        # Fallback
        search:
          - corp.example.com
```

---

## Apply and Verify

```bash
# Apply changes
sudo netplan apply

# Verify active DNS settings
resolvectl status

# Check which DNS is being used for a domain
resolvectl query example.com

# Test DNS resolution
nslookup example.com
dig @1.1.1.1 example.com
```

---

## Set DNS for WiFi

```yaml
network:
  version: 2
  wifis:
    wlan0:
      dhcp4: true
      access-points:
        "MyNetwork":
          password: "mypassword"
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

---

## Debug DNS Issues

```bash
# Check systemd-resolved status
systemctl status systemd-resolved

# View /etc/resolv.conf (should be a symlink)
ls -la /etc/resolv.conf
cat /etc/resolv.conf

# Restart resolver
sudo systemctl restart systemd-resolved
```

---

## Summary

Add a `nameservers` block under your interface in `/etc/netplan/01-netcfg.yaml` with `addresses` (DNS server IPs) and optionally `search` domains. Set `dhcp4-overrides.use-dns: false` to ignore DHCP-assigned DNS. Run `sudo netplan apply` and verify with `resolvectl status`.
