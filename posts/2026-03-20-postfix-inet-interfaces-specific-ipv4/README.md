# How to Set Postfix inet_interfaces to Listen on Specific IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Postfix, IPv4, Inet_interfaces, SMTP, Mail Server, Configuration

Description: Configure the Postfix inet_interfaces parameter to control which IPv4 addresses the SMTP server listens on for incoming mail connections.

## Introduction

`inet_interfaces` in Postfix controls which network interfaces the SMTP server listens on. By default it's often set to `all`, which binds to every interface. Limiting it to specific IPv4 addresses improves security on multi-homed servers.

## Configuring inet_interfaces

```bash
# /etc/postfix/main.cf

# Listen only on specific IPv4 address

inet_interfaces = 203.0.113.10

# Listen on multiple specific IPs
inet_interfaces = 203.0.113.10, 10.0.0.1

# Listen on loopback only (for local delivery only)
inet_interfaces = localhost, 127.0.0.1

# Listen on all interfaces (default)
inet_interfaces = all
```

Reload Postfix after changes:

```bash
sudo postfix check
sudo postfix reload
```

## Understanding inet_interfaces Values

| Value | Behavior |
|---|---|
| `all` | All network interfaces |
| `loopback-only` | `127.0.0.1` and `::1` only |
| `localhost` | Same as loopback-only |
| `203.0.113.10` | Specific IPv4 address |
| `203.0.113.10, 10.0.0.1` | Two specific IPv4 addresses |

## Combining with mynetworks and inet_protocols

```bash
# /etc/postfix/main.cf

# Listen on specific IPv4 only
inet_interfaces = 203.0.113.10

# IPv4 only (critical on dual-stack servers)
inet_protocols = ipv4

# Trust these networks for relay
mynetworks = 127.0.0.1/8, 10.0.0.0/8, 203.0.113.0/24

# Outbound IP
smtp_bind_address = 203.0.113.10
```

## Separate Internal and External SMTP Listeners

For multi-homed servers with separate internal submission and external MX:

```bash
# /etc/postfix/main.cf

# Listen on both public MX and internal submission interfaces
inet_interfaces = 203.0.113.10, 10.0.0.1

# Public interface: port 25 (handled via master.cf)
# Internal: port 587 submission
```

```bash
# /etc/postfix/master.cf

# Port 25 on public IP
smtp      inet  n       -       y       -       -       smtpd
    -o smtpd_client_restrictions=permit_all

# Port 587 submission on internal IP
10.0.0.1:587  inet  n       -       n       -       -       smtpd
    -o syslog_name=postfix/submission
    -o smtpd_tls_security_level=encrypt
    -o smtpd_sasl_auth_enable=yes
```

## Verifying Listen Addresses

```bash
# Check Postfix is listening on correct IPs
sudo ss -tlnp | grep master
# or
sudo ss -tlnp | grep :25

# Expected:
# LISTEN 0 100 203.0.113.10:25 0.0.0.0:*  users:(("master",...))

# Check Postfix effective configuration
postconf inet_interfaces
postconf inet_protocols

# Test SMTP connection to specific IP
telnet 203.0.113.10 25
# Should get Postfix banner
```

## Conclusion

`inet_interfaces` controls which IPv4 addresses Postfix's SMTP server listens on. Set it to specific IPs rather than `all` on multi-homed servers to prevent unintended exposure on internal interfaces. Always pair with `inet_protocols = ipv4` on dual-stack systems to avoid IPv6 listeners, and run `postfix check` before `postfix reload` to validate configuration.
