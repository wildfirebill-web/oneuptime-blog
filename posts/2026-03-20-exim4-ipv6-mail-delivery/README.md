# How to Configure Exim4 for IPv6 Mail Delivery

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Exim4, IPv6, Email, SMTP, Mail Server, Debian, Linux

Description: Configure Exim4 on Debian/Ubuntu to send and receive email over IPv6 by adjusting listener addresses, routing settings, and outbound interface binding.

## Introduction

Exim4 is the default MTA on Debian-based systems. It supports IPv6 but requires specific configuration in `/etc/exim4/update-exim4.conf.conf` and the main configuration to fully enable IPv6 sending and receiving.

## Checking Current Exim4 IPv6 Status

```bash
# Check what addresses Exim is listening on

ss -tlnp | grep exim

# Check current Exim configuration
exim4 -bP local_interfaces
exim4 -bP daemon_smtp_ports
```

## Enabling IPv6 Listening in Exim4

The main configuration file for Debian's Exim4 is:

```bash
sudo nano /etc/exim4/update-exim4.conf.conf
```

Key settings for IPv6:

```ini
# Listen on all interfaces including IPv6 (use :: for all)
dc_local_interfaces='0.0.0.0 ; ::'

# Or listen on specific addresses
dc_local_interfaces='127.0.0.1 ; ::1 ; 2001:db8::10'

# Enable both IPv4 and IPv6 for outbound
dc_minimaldns='false'
```

After editing, regenerate the Exim configuration:

```bash
sudo update-exim4.conf
sudo systemctl restart exim4

# Verify Exim is now listening on IPv6
ss -tlnp | grep 25
# Should show :::25
```

## Configuring Outbound IPv6 Delivery

To bind outbound SMTP connections to a specific IPv6 address, edit the router/transport configuration:

```bash
sudo nano /etc/exim4/conf.d/transport/30_exim4-config_remote_smtp
```

Add the `interface` option to the `remote_smtp` transport:

```exim
remote_smtp:
  driver = smtp
  # Bind outbound connections to this IPv6 address
  interface = <; 2001:db8::10>
  # Enable TLS
  tls_verify_certificates = /etc/ssl/certs/ca-certificates.crt
```

After editing:

```bash
sudo update-exim4.conf
sudo systemctl restart exim4
```

## Configuring exim4 for Split Configuration (Debian)

On Debian, if using split configuration (`dc_use_split_config='true'`):

```bash
# Ensure the listener file includes IPv6
sudo tee /etc/exim4/conf.d/main/00_local_macros << 'EOF'
MAIN_LOCAL_INTERFACES = 0.0.0.0 ; ::
EOF

sudo update-exim4.conf
sudo systemctl restart exim4
```

## Testing IPv6 Mail Delivery

Send a test message using Exim's built-in test mode:

```bash
# Test delivery to an external address
echo "IPv6 Exim test" | exim -v recipient@gmail.com

# Check the mail log for IPv6 connection details
sudo tail -f /var/log/exim4/mainlog | grep -E "::|IPv6"

# Force delivery via IPv6 using specific interface
exim -odf -d -v recipient@example.com
```

## Exim4 and IPv6 DNS Resolution

Ensure Exim4 can resolve AAAA records for outbound routing:

```bash
# Test Exim DNS lookup for a domain
exim4 -bt recipient@example.com

# Check if Exim resolves AAAA records
dig AAAA alt1.gmail-smtp-in.l.google.com
```

## Monitoring Exim IPv6 Delivery in Logs

```bash
# Watch for IPv6 connections in Exim mainlog
sudo grep -E "IPv6|\[2[0-9a-f]+:" /var/log/exim4/mainlog | tail -20

# Monitor the queue
exim4 -bp | head -20

# Check the retry queue for IPv6 delivery failures
exim4 -bpr | grep -i "ipv6\|::"
```

## Troubleshooting

**Exim not listening on IPv6**: Verify `dc_local_interfaces` contains `::` and run `sudo update-exim4.conf && sudo systemctl restart exim4`.

**Outbound delivery uses IPv4**: Check that `interface` is set in the remote_smtp transport and the IPv6 address is configured on the host.

**EHLO name mismatch**: Set `primary_hostname` in Exim config to match the PTR record for your IPv6 address.

## Conclusion

Configuring Exim4 for IPv6 on Debian involves setting `dc_local_interfaces` to include `::` for listening and configuring the `interface` option in the remote_smtp transport for outbound binding. Always verify with `ss` and mail logs that IPv6 is actually being used.
