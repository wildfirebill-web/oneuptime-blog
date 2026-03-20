# How to Set Up Exim to Bind to a Specific IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Exim, IPv4, Email, SMTP, Configuration, Linux, Binding

Description: Learn how to configure Exim to listen and send email on a specific IPv4 address using the local_interfaces and smtp_bind_address directives.

---

On multi-homed servers with multiple IPv4 addresses, you may need Exim to listen on and originate connections from a specific IP — for instance, to ensure email comes from an IP with proper PTR/SPF records.

## Configuring the Listening Address

The `local_interfaces` option in Exim's main configuration controls which IP:port combinations Exim listens on.

```
# /etc/exim4/exim4.conf.template (or /etc/exim4/conf.d/main/02_exim4-config_options)

# Listen only on a specific IPv4 address on port 25
local_interfaces = 192.168.1.10:25

# Listen on multiple addresses (separated by colon-delimited list)
# local_interfaces = 192.168.1.10:25 : 127.0.0.1:25

# Listen on all IPv4 interfaces on port 25 (no IPv6)
# local_interfaces = 0.0.0.0:25
```

## Binding Outbound Connections to a Specific IPv4 Address

The `smtp_bind_address` option sets the source IP for outgoing SMTP connections.

```
# /etc/exim4/exim4.conf.template

# All outbound SMTP connections originate from this IPv4 address
smtp_bind_address = 203.0.113.10
```

This ensures Exim uses the correct source IP for SPF and PTR record lookups.

## Debian/Ubuntu: Split Configuration

On Debian-based systems, Exim uses a split configuration in `/etc/exim4/conf.d/`. Add the directives to the appropriate file.

```bash
# Edit the main options file
nano /etc/exim4/conf.d/main/02_exim4-config_options
```

```
# Add or modify these lines:
local_interfaces = 192.168.1.10:25 : 127.0.0.1:25
smtp_bind_address = 192.168.1.10
```

```bash
# Regenerate the combined config from split files
update-exim4.conf

# Test the configuration
exim -bV

# Restart Exim to apply changes
systemctl restart exim4
```

## Disabling IPv6 in Exim

```
# /etc/exim4/exim4.conf.template

# Prevent Exim from creating IPv6 listening sockets
# Set local_interfaces to only IPv4 addresses
local_interfaces = 0.0.0.0:25

# Prevent outbound IPv6 connections
# (Exim will only attempt IPv4 connections for deliveries)
dns_ipv4_lookup = *    # Prefer IPv4 for all DNS lookups
```

## Verifying the Configuration

```bash
# Check which ports Exim is listening on
ss -tlnp | grep exim

# Test SMTP connection to the specific IPv4 address
telnet 192.168.1.10 25

# Send a test email
echo "Test" | exim -v recipient@example.com

# Check the mail log
tail -f /var/log/exim4/mainlog
```

## Key Takeaways

- `local_interfaces` controls which IPv4:port pairs Exim listens on; specify explicit IPs to avoid binding to IPv6.
- `smtp_bind_address` forces all outbound SMTP connections to originate from a specific IPv4 address.
- On Debian, edit `/etc/exim4/conf.d/main/` files and run `update-exim4.conf` to regenerate the combined config.
- Set `dns_ipv4_lookup = *` to prefer A records over AAAA records for all DNS lookups.
