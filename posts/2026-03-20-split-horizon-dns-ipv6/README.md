# How to Configure Split-Horizon DNS with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Split-Horizon DNS, IPv6, BIND, Unbound, Internal DNS, View

Description: Configure split-horizon DNS to return different AAAA records for internal and external clients, using BIND views or Unbound stub/forward zones.

## Introduction

Split-horizon (split-view) DNS returns different answers for the same domain based on the requester's network. Internal IPv6 clients receive private AAAA addresses, while external clients get public AAAA addresses. This is essential for services that have both internal and external IPv6 addresses.

## Architecture

```text
Internal IPv6 client (2001:db8:internal::/48)
    → DNS query: www.example.com AAAA
    → Split-horizon DNS
    → Returns: fd00::10 (private ULA address)

External client
    → DNS query: www.example.com AAAA
    → Split-horizon DNS
    → Returns: 2001:db8::10 (public IPv6 address)
```

## Option 1: BIND Views

```nginx
# /etc/bind/named.conf

acl "internal-v6" {
    2001:db8:internal::/48;
    fd00::/8;
    ::1;
    127.0.0.0/8;
    192.168.0.0/16;
};

view "internal" {
    match-clients { "internal-v6"; };

    zone "example.com" {
        type master;
        file "/etc/bind/zones/internal/db.example.com";
    };

    # Recursion for internal clients
    recursion yes;
    allow-query { "internal-v6"; };
};

view "external" {
    match-clients { any; };

    zone "example.com" {
        type master;
        file "/etc/bind/zones/external/db.example.com";
    };

    recursion no;
    allow-query { any; };
};
```

```dns-zone
; /etc/bind/zones/internal/db.example.com
www     IN  AAAA    fd00::10       ; Private ULA address
api     IN  AAAA    fd00::11
db      IN  AAAA    fd00::12
```

```dns-zone
; /etc/bind/zones/external/db.example.com
www     IN  AAAA    2001:db8::10   ; Public IPv6 address
; api and db are not exposed externally
```

## Option 2: Unbound with Stub Zones

```yaml
# /etc/unbound/unbound.conf

server:
    interface: ::0
    access-control: ::/0 allow
    prefer-ip6: yes

# Internal zone served by internal authoritative server

stub-zone:
    name: "example.com"
    stub-addr: fd00::53   # Internal authoritative server
    stub-first: no

# External zone - resolved from authoritative servers
# (no override needed - recursive resolver handles it)
```

## Option 3: CoreDNS with View Plugin

```corefile
# /etc/coredns/Corefile

example.com.:53 {
    view internal {
        expr incidr(client_ip(), "2001:db8:internal::/48") ||
             incidr(client_ip(), "fd00::/8")
    }
    file /etc/coredns/zones/example.com.internal
    log
}

example.com.:53 {
    view external
    file /etc/coredns/zones/example.com.external
    log
}
```

## Testing Split-Horizon

```bash
# Test from internal IPv6 address
dig AAAA www.example.com @2001:db8::53 \
    -b 2001:db8:internal::1
# Expected: fd00::10

# Test from external
dig AAAA www.example.com @2001:db8::53 \
    -b 2001:db8:external::1
# Expected: 2001:db8::10

# Test from loopback
dig AAAA www.example.com @::1
# Expected: fd00::10 (loopback is internal)
```

## Split Horizon for Kubernetes Services

```yaml
# CoreDNS ConfigMap for Kubernetes split-horizon
# Internal: resolve service.namespace.svc.cluster.local to ClusterIP
# External: resolve to LoadBalancer IPv6 address

# coredns ConfigMap
data:
  Corefile: |
    example.com:53 {
        view k8s-internal {
            expr incidr(client_ip(), "fd00::/8")
        }
        # Serve from cluster-internal zone file
        file /etc/coredns/internal-zones/example.com
    }
    example.com:53 {
        view external
        forward . 2606:4700:4700::1111
    }
```

## Conclusion

Split-horizon DNS with IPv6 uses the same mechanisms as IPv4: BIND views match on IPv6 ACLs, Unbound uses stub zones, and CoreDNS uses the view plugin. The key difference is using IPv6 prefixes (fd00::/8 for ULA, specific global prefixes for external) in ACL definitions. Use OneUptime to verify that internal and external DNS return the expected AAAA addresses.
