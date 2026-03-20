# How to Use iptables-persistent to Survive Reboots on Debian

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: iptables, Debian, Linux, Firewall, Persistence, Iptables-persistent

Description: Configure iptables-persistent on Debian to automatically restore iptables and ip6tables rules at boot, ensuring your firewall configuration survives system restarts.

On Debian and Debian-based systems, `iptables-persistent` is the standard package for making iptables rules survive reboots. It provides a systemd service (`netfilter-persistent`) that restores saved rules during the boot sequence.

## Install iptables-persistent

```bash
# Update package list

sudo apt update

# Install iptables-persistent
sudo apt install iptables-persistent -y

# During installation, a debconf dialog asks:
# "Save current IPv4 rules? [Yes/No]"
# "Save current IPv6 rules? [Yes/No]"
# Answer Yes to save current rules immediately
```

## Where Rules Are Stored

```bash
# IPv4 rules
cat /etc/iptables/rules.v4

# IPv6 rules
cat /etc/iptables/rules.v6

# These files are automatically restored at boot by:
# /lib/systemd/system/netfilter-persistent.service
```

## Save Rules After Changes

After making changes, you must explicitly save:

```bash
# Method 1: Direct file save
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6

# Method 2: Using netfilter-persistent
sudo netfilter-persistent save

# Method 3: Using the iptables-save command alias
sudo /etc/init.d/netfilter-persistent save

# Verify save was successful
ls -la /etc/iptables/rules.v4
stat /etc/iptables/rules.v4  # Check modification time
```

## Service Management

```bash
# Check service status
sudo systemctl status netfilter-persistent

# Enable service (should be auto-enabled after install)
sudo systemctl enable netfilter-persistent

# Manually restore rules (without reboot)
sudo systemctl restart netfilter-persistent
# or
sudo netfilter-persistent reload

# Stop and remove all rules (dangerous on remote server!)
sudo systemctl stop netfilter-persistent
```

## Verify Rules at Boot

Test that rules load correctly on boot:

```bash
# Simulate a reboot by flushing and restoring
sudo iptables -F   # Flush all rules
sudo iptables -P INPUT ACCEPT   # Reset policies

# Restore as service would
sudo netfilter-persistent reload

# Verify rules are back
sudo iptables -L -n -v --line-numbers
```

## The Complete Workflow

```bash
# 1. Set up your rules
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -j DROP
sudo iptables -P INPUT DROP

# 2. Test (verify SSH still works, web server reachable)

# 3. Save permanently
sudo netfilter-persistent save

# 4. Verify the save
cat /etc/iptables/rules.v4

# 5. Test reboot restoration
sudo systemctl restart netfilter-persistent

# 6. Confirm rules loaded
sudo iptables -L -n -v
```

## Troubleshoot: Rules Not Loading After Reboot

```bash
# Check if service is enabled
sudo systemctl is-enabled netfilter-persistent

# Check service logs
sudo journalctl -u netfilter-persistent

# Check rules file exists and has content
ls -la /etc/iptables/rules.v4
wc -l /etc/iptables/rules.v4  # Should have multiple lines

# If file is empty or missing, rules were never saved
# Create an empty valid file and add rules:
echo -e "*filter\n:INPUT ACCEPT [0:0]\n:FORWARD ACCEPT [0:0]\n:OUTPUT ACCEPT [0:0]\nCOMMIT" \
  | sudo tee /etc/iptables/rules.v4
```

`iptables-persistent` is the simplest and most reliable way to make iptables survive reboots on Debian systems - it integrates cleanly with systemd and requires no manual boot scripts.
