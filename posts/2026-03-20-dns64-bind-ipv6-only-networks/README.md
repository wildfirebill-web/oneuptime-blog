# How to Configure DNS64 with BIND for IPv6-Only Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS64, BIND, IPv6, NAT64, IPv6-Only, Networking

Description: Configure BIND DNS server as DNS64 to synthesize AAAA records for IPv4-only domains, enabling IPv6-only clients to reach IPv4 services through NAT64.

## Introduction

DNS64 is a DNS server feature that synthesizes AAAA records when a domain has only A records. It maps IPv4 addresses into the NAT64 prefix (64:ff9b::/96 or a custom prefix), allowing IPv6-only clients to discover the translated address and connect via NAT64.

## How DNS64 Works

```
IPv6-only client requests: AAAA google.com
DNS64 server:
  1. Queries upstream for AAAA google.com → no result
  2. Queries upstream for A google.com    → 172.217.0.0
  3. Synthesizes: 64:ff9b::172.217.0.0   (NAT64 prefix + IPv4)
  4. Returns AAAA 64:ff9b::ac:d9:0:0
Client connects to 64:ff9b::ac:d9:0:0 → NAT64 → 172.217.0.0
```

## BIND Configuration

```bash
# /etc/bind/named.conf.options

options {
    directory "/var/cache/bind";

    # DNS64 configuration
    dns64 64:ff9b::/96 {
        # Which clients get DNS64 synthesis
        clients { any; };

        # Excluded prefixes (no synthesis for these — they have real AAAA)
        # exclude { ::ffff:0:0/96; 64:ff9b::/96; };

        # Mapped: apply synthesis only to these IPv4 ranges
        # mapped { !RFC1918-RANGES; any; };
        # Uncomment above to skip NAT64 for RFC1918 addresses

        break-dnssec yes;    # Allow DNSSEC-signed records to be synthesized
    };

    # Forwarders for upstream resolution
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };

    forward only;

    # Allow queries from IPv6-only clients
    allow-query { any; };
    recursion yes;
};
```

## Exclude RFC1918 from Synthesis

```bash
# /etc/bind/named.conf.options

# Define RFC1918 ACL
acl "rfc1918" {
    10.0.0.0/8;
    172.16.0.0/12;
    192.168.0.0/16;
};

options {
    dns64 64:ff9b::/96 {
        clients { any; };
        mapped { !rfc1918; any; };   # Don't synthesize for private IPs
        break-dnssec yes;
    };
    forwarders { 8.8.8.8; 8.8.4.4; };
    forward only;
    recursion yes;
};
```

## Starting BIND

```bash
sudo systemctl restart bind9  # or named

# Verify BIND is listening
sudo ss -ulnp | grep :53
sudo ss -tlnp | grep :53

# Check BIND logs
sudo journalctl -u bind9 -f
```

## Testing DNS64

```bash
# From DNS64 server itself:
dig AAAA ipv4only.arpa @127.0.0.1
# Should return: 64:ff9b::c000:200 (synthesized for 192.0.2.0)

# For a real IPv4-only domain (example):
dig AAAA example.com @127.0.0.1
# If example.com has only A record:
# Returns AAAA 64:ff9b::<ipv4_in_hex>

# Compare with real DNS:
dig A example.com @8.8.8.8      # Returns real IPv4
dig AAAA example.com @127.0.0.1 # Returns synthesized IPv6

# Test from IPv6-only client (set DNS to DNS64 server):
# /etc/resolv.conf
# nameserver 2001:db8::dns64server
```

## Client Configuration

```bash
# Point IPv6-only clients to the DNS64 server
# /etc/resolv.conf
nameserver 2001:db8::5    # Your DNS64 server IPv6 address

# Or via DHCPv6 option 23 (dns-servers)
```

## Conclusion

BIND's `dns64` directive synthesizes AAAA records using the NAT64 prefix when only A records exist. Configure `dns64 64:ff9b::/96 { clients { any; }; }` and set `break-dnssec yes` to allow synthesis of DNSSEC-signed records. Exclude RFC1918 addresses from synthesis using the `mapped` option. Test with `dig AAAA` against the DNS64 server and verify synthesized addresses use the `64:ff9b::` prefix.
