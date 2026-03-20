# How to Set Up AD with FQDN vs IP Address in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Active Directory, LDAP, DNS, Networking

Description: Understand when to use FQDN versus IP address for Active Directory connections in Portainer and how DNS affects LDAP over TLS.

---

When configuring Active Directory authentication in Portainer, you must decide whether to connect using a domain controller's FQDN or its IP address. This choice has significant implications for TLS validation and DNS failover.

## Why FQDN is Strongly Recommended

Using the FQDN of the domain controller (`dc01.corp.example.com`) rather than an IP address is required when:
- Using LDAPS (LDAP over TLS) with valid certificates
- The AD SSL certificate contains the hostname in its Subject Alternative Name
- You want DNS-based failover to replica domain controllers

```bash
# CORRECT: Using FQDN with LDAPS

# The AD SSL cert is issued to dc01.corp.example.com
# TLS validation succeeds because hostname matches the cert
curl -X PUT https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "LDAPSettings": {
      "URLs": ["ldaps://dc01.corp.example.com:636"],
      "TLSConfig": {"TLS": true, "TLSSkipVerify": false}
    }
  }' --insecure

# PROBLEMATIC: Using IP address with LDAPS
# TLS validation FAILS because cert was issued to the hostname, not the IP
# (unless the cert also contains the IP in SANs)
# "LDAPSettings": {"URLs": ["ldaps://192.168.1.10:636"]}
```

## When IP Addresses Work

IP addresses are acceptable only when:
- Using plain LDAP (port 389, unencrypted) - not recommended for production
- The SSL certificate includes the IP address as a Subject Alternative Name
- You've set `TLSSkipVerify: true` - only appropriate for testing

```bash
# Using IP with TLSSkipVerify (development/testing only)
curl -X PUT https://localhost:9443/api/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "LDAPSettings": {
      "URLs": ["ldaps://192.168.1.10:636"],
      "TLSConfig": {"TLS": true, "TLSSkipVerify": true}
    }
  }' --insecure
# WARNING: TLSSkipVerify disables certificate validation - not for production
```

## DNS Resolution inside the Container

Portainer runs inside Docker. The Portainer container must be able to resolve the FQDN:

```bash
# Test DNS resolution from inside the Portainer container
docker exec portainer nslookup dc01.corp.example.com

# If DNS fails, add a custom DNS server to Docker
docker stop portainer && docker container rm portainer

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  --dns 192.168.1.5 \           # Corporate DNS server
  --dns-search corp.example.com \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## Using a DNS Alias (CNAME) for Resilience

Point Portainer at a CNAME that can be updated to switch between DCs:

```bash
# In your DNS: ldap.corp.example.com -> dc01.corp.example.com
# If DC1 fails, update the CNAME to point to DC2
# Portainer configuration never needs to change

# Configure Portainer with the CNAME
"URLs": ["ldaps://ldap.corp.example.com:636"]
```

## Multi-DC Configuration

For AD environments with multiple domain controllers:

```bash
# Configure multiple DC URLs for automatic failover
"LDAPSettings": {
  "URLs": [
    "ldaps://dc01.corp.example.com:636",
    "ldaps://dc02.corp.example.com:636",
    "ldaps://dc03.corp.example.com:636"
  ]
}
```

## Verify Hostname Resolution

```bash
# From the Docker host, verify DNS works
host dc01.corp.example.com

# From inside the container
docker exec portainer /bin/sh -c "nslookup dc01.corp.example.com"

# Check the certificate SAN for the IP (if needed)
openssl s_client -connect dc01.corp.example.com:636 </dev/null 2>/dev/null | \
  openssl x509 -noout -text | grep -A1 "Subject Alternative Name"
```

---

*Monitor your Active Directory and Portainer services with [OneUptime](https://oneuptime.com).*
