# How to Configure DNS64 with Unbound

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, DNS64, Unbound, NAT64, DNS Configuration

Description: Learn how to enable DNS64 in the Unbound recursive resolver to synthesize AAAA records for IPv4-only domains, supporting IPv6-only clients through a NAT64 gateway.

## Prerequisites

- Unbound 1.5.4 or later (DNS64 support added in 1.5.4)
- Root access on the DNS server
- A configured NAT64 gateway using the same prefix

## Installing Unbound

```bash
# Ubuntu/Debian

apt update && apt install unbound

# RHEL/CentOS/Fedora
dnf install unbound

# Verify version supports DNS64
unbound -V | grep Version
```

## Unbound Configuration for DNS64

Unbound's DNS64 configuration is done in the `module-config` directive and a dedicated `dns64` section in `unbound.conf`:

```yaml
# /etc/unbound/unbound.conf

server:
    # Listen on all interfaces including IPv6
    interface: 0.0.0.0
    interface: ::0

    # Allow queries from local networks
    access-control: 192.168.0.0/16 allow
    access-control: 2001:db8::/32 allow
    access-control: ::1 allow
    access-control: 127.0.0.1 allow

    # Enable DNSSEC validation (DNS64 is compatible with this)
    auto-trust-anchor-file: "/var/lib/unbound/root.key"

    # Module configuration: dns64 must come before validator and iterator
    module-config: "dns64 validator iterator"

# DNS64 configuration block
dns64:
    # The NAT64 prefix to use for synthesis
    # Must match your NAT64 gateway's prefix
    prefix: 64:ff9b::/96

    # Exclude these IPv4 ranges from synthesis (RFC 1918 and loopback)
    # These will not get synthesized AAAA records
    # dns64-synthall is off by default, only domains without AAAA get synthesized
```

## Advanced DNS64 Configuration Options

```yaml
dns64:
    # The NAT64 prefix
    prefix: 64:ff9b::/96

    # If set to yes, synthesize records even when AAAA exists
    # Default: no (only synthesize when no native AAAA exists)
    # dns64-synthall: no
```

## Restricting DNS64 to Specific Clients

Unbound does not natively support per-client DNS64 prefixes in the same way BIND does, but you can run multiple Unbound instances on different ports or use views. For most deployments, DNS64 applies globally to all clients querying that resolver.

For fine-grained control, use a separate DNS64-enabled Unbound instance for IPv6-only clients:

```bash
# Create a separate config for the DNS64 instance
cp /etc/unbound/unbound.conf /etc/unbound/unbound-dns64.conf

# Edit the DNS64 instance to listen on a different port or address
# e.g., only listen on the IPv6-only subnet interface
```

## Checking the Configuration

```bash
# Validate the Unbound configuration
unbound-checkconf /etc/unbound/unbound.conf

# Restart Unbound to apply changes
systemctl restart unbound

# Check service status
systemctl status unbound
```

## Testing DNS64 with dig

```bash
# Query the Unbound DNS64 resolver for a domain with only an A record
# Replace 127.0.0.1 with the Unbound server's address if remote
dig AAAA example.com @127.0.0.1

# Expected output (example.com has IPv4 93.184.216.34)
# example.com. 60 IN AAAA 64:ff9b::5db8:d822

# Confirm that real AAAA records pass through unchanged
dig AAAA google.com @127.0.0.1

# Test with a domain known to have only A records
dig AAAA ipv4only.arpa @127.0.0.1
```

## Verifying Module Order

The module order in `module-config` is critical. DNS64 must process queries before validation:

```yaml
# Correct order for DNS64 + DNSSEC validation
module-config: "dns64 validator iterator"

# If you don't use DNSSEC validation (not recommended):
module-config: "dns64 iterator"
```

## Monitoring Unbound DNS64 Statistics

```bash
# Enable statistics in unbound.conf
# server:
#     statistics-interval: 60
#     extended-statistics: yes

# View live statistics
unbound-control stats_noreset | grep -i dns64

# Check query counts
unbound-control stats | grep num.queries
```

## Common Issues and Fixes

**Issue**: Synthesized AAAA records not returned
- Verify `dns64` appears before `validator` in `module-config`
- Confirm prefix matches NAT64 gateway

**Issue**: DNSSEC validation failures for synthesized records
- Unbound handles this correctly by not synthesizing DNSSEC-signed responses when validation is enabled

**Issue**: Unbound crashes after adding dns64
- Ensure Unbound version is 1.5.4 or newer
- Check logs: `journalctl -u unbound`

## Summary

Enabling DNS64 in Unbound requires adding the `dns64` module to `module-config` (before `validator`) and configuring the NAT64 prefix in the `dns64` section. The configuration is simpler than BIND's but equally effective for synthesizing AAAA records for IPv4-only domains.
