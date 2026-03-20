# How to Restrict Access to Nginx by IPv4 CIDR Range

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Nginx, IPv4, CIDR, Access Control, Security, Networking

Description: Restrict access to Nginx virtual hosts and locations using IPv4 CIDR notation to allow or deny entire network ranges with precise subnet masks.

## Introduction

CIDR (Classless Inter-Domain Routing) notation lets you specify IP ranges concisely. Instead of listing every IP in a /24 subnet, you write `192.168.1.0/24`. Nginx natively supports CIDR in its `allow` and `deny` directives, making network-level access control precise and maintainable.

## Understanding CIDR Notation in Nginx

| CIDR | Range | Hosts |
|---|---|---|
| `10.0.0.0/8` | 10.0.0.0 – 10.255.255.255 | 16,777,214 |
| `172.16.0.0/12` | 172.16.0.0 – 172.31.255.255 | 1,048,574 |
| `192.168.1.0/24` | 192.168.1.0 – 192.168.1.255 | 254 |
| `10.10.5.0/28` | 10.10.5.0 – 10.10.5.15 | 14 |
| `203.0.113.64/26` | 203.0.113.64 – 203.0.113.127 | 62 |

## Restricting Access by CIDR Range

```nginx
# /etc/nginx/conf.d/cidr-access.conf

server {
    listen 80;
    server_name internal.example.com;

    location / {
        # Allow entire corporate network (RFC 1918 private ranges)
        allow 10.0.0.0/8;
        allow 172.16.0.0/12;
        allow 192.168.0.0/16;

        # Allow specific public CIDR (e.g., remote office)
        allow 203.0.113.64/26;

        # Deny everything else
        deny all;

        proxy_pass http://internal_backend;
    }
}
```

## Allowing Only a Specific Department's Subnet

```nginx
server {
    listen 80;
    server_name finance.example.com;

    location / {
        # Only the finance VLAN (10.10.5.0/24) can access
        allow 10.10.5.0/24;

        # And the IT admin subnet
        allow 10.0.1.0/28;   # Only 14 hosts in this /28

        deny all;

        proxy_pass http://finance_app;
    }
}
```

## Using geo Module for Complex CIDR Logic

For conditional logic based on CIDR membership, the `geo` module is more flexible than chained allow/deny:

```nginx
http {
    geo $remote_addr $is_trusted_network {
        default          0;

        10.0.0.0/8       1;   # Internal RFC-1918
        172.16.0.0/12    1;
        192.168.0.0/16   1;
        203.0.113.64/26  1;   # Remote office CIDR
    }

    server {
        listen 80;

        location /admin {
            # Deny if not in trusted network
            if ($is_trusted_network = 0) {
                return 403 "Access restricted to trusted networks";
            }
            proxy_pass http://admin_backend;
        }
    }
}
```

## Splitting Traffic by CIDR for Multi-Tenant Setups

Route different CIDR ranges to different backends:

```nginx
http {
    geo $remote_addr $tenant_backend {
        default            "shared_backend";
        10.1.0.0/16        "tenant_a_backend";
        10.2.0.0/16        "tenant_b_backend";
        10.3.0.0/16        "tenant_c_backend";
    }

    upstream shared_backend   { server 192.168.100.10:8080; }
    upstream tenant_a_backend { server 192.168.100.20:8080; }
    upstream tenant_b_backend { server 192.168.100.30:8080; }
    upstream tenant_c_backend { server 192.168.100.40:8080; }

    server {
        listen 80;
        location / {
            proxy_pass http://$tenant_backend;
        }
    }
}
```

## Verifying CIDR Rules

Test that your CIDR rules are working as expected:

```bash
# Test from an IP within the allowed range (use curl --interface)

curl --interface 10.10.5.50 http://finance.example.com/

# Test from an IP outside the range
curl --interface 203.0.113.200 http://finance.example.com/
# Expected: HTTP 403

# Check Nginx error log for access denials
sudo tail -f /var/log/nginx/error.log | grep "access forbidden"
```

## Conclusion

Nginx CIDR-based access control is expressed in a few lines of configuration and requires no external modules. Combine `allow`/`deny` for simple rules and the `geo` module for conditional logic. Always audit your CIDR ranges when network topology changes to avoid accidentally blocking legitimate users or leaving gaps in access control.
