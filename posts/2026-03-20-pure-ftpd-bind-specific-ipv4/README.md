# How to Set Up Pure-FTPd to Bind to a Specific IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Pure-FTPd, FTP, IPv4, Configuration, Linux, Binding, Networking

Description: Learn how to configure Pure-FTPd to listen only on a specific IPv4 address, useful on multi-homed servers with multiple IP addresses.

---

On servers with multiple IPv4 addresses, binding Pure-FTPd to a specific IP ensures FTP traffic is segregated from other services. This is configured differently depending on whether Pure-FTPd was installed via the standalone binary or a configuration file.

## Method 1: Command-Line Binding

Pure-FTPd accepts the bind address as a command-line argument.

```bash
# Bind to a specific IPv4 address and port
# Format: --bind <ip>,<port>
pure-ftpd --bind 192.168.1.10,21 &

# Or with common security options
pure-ftpd \
  --bind 192.168.1.10,21 \       # Bind to specific IPv4
  --noanonymous \                 # Disable anonymous access
  --createhomedir \               # Create home dirs automatically
  --minuid 1000 \                 # Minimum UID allowed to log in
  --passiveportrange 40000:50000 \ # Passive mode port range
  --daemonize                     # Run as a daemon
```

## Method 2: Debian/Ubuntu Configuration Files

On Debian-based systems, Pure-FTPd uses per-option configuration files in `/etc/pure-ftpd/conf/`.

```bash
# Bind to a specific IPv4 address
echo "192.168.1.10" > /etc/pure-ftpd/conf/ForcePassiveIP

# Or set the bind address
echo "192.168.1.10 21" > /etc/pure-ftpd/conf/Bind
```

```bash
# Disable anonymous access
echo "yes" > /etc/pure-ftpd/conf/NoAnonymous

# Set passive port range
echo "40000 50000" > /etc/pure-ftpd/conf/PassivePortRange

# Restart to apply
systemctl restart pure-ftpd
```

## Method 3: SystemD Service Override

Modify the systemd service to pass the bind address.

```ini
# /etc/systemd/system/pure-ftpd.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/sbin/pure-ftpd \
  --bind 192.168.1.10,21 \
  --noanonymous \
  --passiveportrange 40000:50000
```

```bash
systemctl daemon-reload
systemctl restart pure-ftpd
```

## Verifying the Binding

```bash
# Confirm Pure-FTPd is listening on the specific IPv4 address
ss -tlnp | grep pure-ftpd
# Expected: LISTEN 192.168.1.10:21

# Test connection from a client
ftp 192.168.1.10

# Should fail from the other IP (not bound)
ftp 192.168.1.20
```

## Passive Mode and NAT

If the server is behind NAT, set `ForcePassiveIP` to the public IP:

```bash
# Advertise the public IPv4 in PASV responses
echo "203.0.113.10" > /etc/pure-ftpd/conf/ForcePassiveIP
systemctl restart pure-ftpd
```

## Key Takeaways

- Use `--bind <ip>,<port>` on the command line to bind Pure-FTPd to a specific IPv4 address.
- On Debian/Ubuntu, write the bind address to `/etc/pure-ftpd/conf/Bind`.
- Set `ForcePassiveIP` to the public IPv4 address when the server is behind NAT.
- Always set `PassivePortRange` and open those ports in the firewall for passive mode to work.
