# How to Set Up GeoDNS with IPv6 AAAA Records

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, GeoDNS, DNS, AAAA Records, Content Delivery

Description: Learn how to configure GeoDNS to return different IPv6 AAAA records based on the client's geographic location, enabling regional traffic routing for dual-stack services.

## What Is GeoDNS?

GeoDNS returns different DNS responses based on the client's geographic location (determined by IP address geolocation). For IPv6, this means serving different AAAA records to users in different regions, directing them to the nearest or most appropriate server.

## GeoDNS with PowerDNS GeoIP Backend

PowerDNS has a native GeoIP backend that supports IPv6 AAAA responses based on client location:

### Install the GeoIP Backend

```bash
# Ubuntu/Debian
apt install pdns-backend-geoip libmaxminddb-dev

# Download MaxMind GeoLite2 database (requires free registration)
# Place database at /etc/powerdns/geoip/GeoLite2-City.mmdb
```

### Configure the GeoIP Backend

```yaml
# /etc/powerdns/geoip.yaml
domains:
  - domain: example.com
    ttl: 300
    records:
      # Root apex
      example.com:
        - soa: ns1.example.com. admin.example.com. 2026032001 7200 1800 604800 300
        - ns:
          - ns1.example.com.
          - ns2.example.com.
      # GeoDNS for www
      www.example.com:
        - aaaa:
            # North America → US datacenter
            "%ci.%co.%cc":
              "*.*.US":
                - "2001:db8:us::1"
              # Europe → EU datacenter
              "*.*.DE":
                - "2001:db8:eu::1"
              "*.*.GB":
                - "2001:db8:eu::1"
              "*.*.FR":
                - "2001:db8:eu::1"
              # Asia-Pacific → AP datacenter
              "*.*.JP":
                - "2001:db8:ap::1"
              "*.*.AU":
                - "2001:db8:ap::1"
              # Default (global)
              "*.*.??":
                - "2001:db8:us::1"
```

### Enable the GeoIP Backend in pdns.conf

```ini
# /etc/powerdns/pdns.conf
launch=geoip
geoip-database-files=/etc/powerdns/geoip/GeoLite2-City.mmdb
geoip-zones-file=/etc/powerdns/geoip.yaml
```

## GeoDNS with BIND Views

BIND's `view` directive allows returning different responses to different client subnets:

```named
// /etc/named.conf - Geographic routing with views

// ACLs for geographic regions
acl "north-america" {
    104.0.0.0/8;    // example NA ranges
    192.0.2.0/24;
};

acl "europe" {
    185.0.0.0/8;    // example EU ranges
    2001:db8:eu::/48;
};

acl "asia-pacific" {
    203.0.113.0/24;  // example APAC ranges
    2001:db8:ap::/48;
};

// View for North America clients
view "north-america" {
    match-clients { north-america; };

    zone "example.com" {
        type master;
        file "/var/named/example.com.na.zone";  // US server AAAA
    };
};

// View for Europe clients
view "europe" {
    match-clients { europe; };

    zone "example.com" {
        type master;
        file "/var/named/example.com.eu.zone";  // EU server AAAA
    };
};

// Default view
view "default" {
    match-clients { any; };

    zone "example.com" {
        type master;
        file "/var/named/example.com.default.zone";
    };
};
```

Example zone files for each region:

```dns
; /var/named/example.com.na.zone - North America
www  300  IN  A     203.0.113.1
www  300  IN  AAAA  2001:db8:us::1

; /var/named/example.com.eu.zone - Europe
www  300  IN  A     198.51.100.1
www  300  IN  AAAA  2001:db8:eu::1
```

## Testing GeoDNS Responses

```bash
# Test from different source IPs using edns-client-subnet
# This passes the client's subnet in EDNS0 for GeoDNS testing
dig AAAA www.example.com @ns1.example.com \
    +subnet=203.0.113.0/24  # Simulate a query from this subnet

# Test from a North American IP
dig AAAA www.example.com @ns1.example.com +subnet=104.1.2.3/32

# Test from a European IP
dig AAAA www.example.com @ns1.example.com +subnet=185.1.2.3/32

# Verify different AAAA records are returned for different regions
```

## Monitoring GeoDNS with OneUptime

Configure regional monitors in OneUptime to verify each datacenter's AAAA response:

1. Create a DNS monitor checking `www.example.com` AAAA from a North American probe
2. Create a DNS monitor checking `www.example.com` AAAA from a European probe
3. Alert if the wrong AAAA is returned for a region

## Summary

GeoDNS with IPv6 AAAA records enables geographic traffic steering by returning different IPv6 addresses based on client location. PowerDNS GeoIP backend provides a YAML-based configuration with MaxMind database support. BIND views offer similar functionality for split-horizon DNS. Use EDNS0 client-subnet with `dig +subnet=` to test different regions without physically being there.
