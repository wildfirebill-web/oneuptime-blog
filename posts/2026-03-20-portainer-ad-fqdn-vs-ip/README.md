# How to Set Up AD with FQDN vs IP Address in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Active Directory, LDAP, DNS, Configuration, TLS

Description: Understand the implications of using FQDN versus IP address for your Active Directory server in Portainer LDAP configuration, particularly for TLS certificates.

## Introduction

When configuring Active Directory authentication in Portainer, you must specify the domain controller's address. You can use either its FQDN (e.g., `dc01.corp.example.com`) or IP address (e.g., `192.168.1.10`). The choice matters significantly when using LDAPS (TLS), where certificate validation depends on the hostname matching.

## FQDN vs IP Address: Key Differences

| Aspect | FQDN | IP Address |
|--------|------|-----------|
| TLS Certificate | Matches if cert has correct CN/SAN | Usually won't match (cert doesn't include IP) |
| DNS Dependency | Requires DNS resolution | No DNS needed |
| Portability | Works across network changes | Breaks if IP changes |
| AD Best Practice | Recommended | Not recommended |

## Recommendation

**Always use FQDN** unless you have a specific reason not to. The FQDN must match the Common Name (CN) or Subject Alternative Name (SAN) in the AD domain controller's certificate.

## Using FQDN (Recommended)

```bash
# Verify DNS resolution from Portainer host
nslookup dc01.corp.example.com
# or
dig dc01.corp.example.com

# Verify from Portainer container
docker exec portainer nslookup dc01.corp.example.com

# Test LDAP connectivity via FQDN
ldapsearch -x \
  -H ldap://dc01.corp.example.com:389 \
  -D "portainer-svc@corp.example.com" \
  -w "password" \
  -b "DC=corp,DC=example,DC=com" \
  -s base "(objectClass=*)"
```

**Portainer configuration with FQDN:**
```
Server: dc01.corp.example.com:389
```

## Using the Domain Name (Load Balanced)

For load balancing across multiple domain controllers, use the domain name itself — Windows DNS SRV records will handle distribution:

```
Server: corp.example.com:389
```

This is the most resilient approach in multi-DC environments.

## Using IP Address (When FQDN Doesn't Resolve)

If DNS is unavailable or unreliable, you may need to use an IP address:

```
Server: 192.168.1.10:389
```

**Important**: When using an IP address with LDAPS, certificate verification will fail unless:
1. The certificate includes the IP in its Subject Alternative Names (rare)
2. You set **Skip TLS Verification** to ON

```bash
# Check if the AD cert includes the IP in SANs
openssl s_client -connect 192.168.1.10:636 < /dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep -A5 "Subject Alternative Name"
```

## Adding a Hosts Entry (Workaround for DNS Issues)

If DNS resolution is the problem, add a hosts entry on the Docker host:

```bash
# Add to /etc/hosts on the Docker host
echo "192.168.1.10 dc01.corp.example.com" | sudo tee -a /etc/hosts

# Add to Portainer container's hosts via compose
services:
  portainer:
    extra_hosts:
      - "dc01.corp.example.com:192.168.1.10"
```

The `extra_hosts` Docker Compose option adds entries to the container's `/etc/hosts` file, enabling FQDN resolution without depending on DNS.

## Certificate Validation Issues

When using LDAPS with FQDN:

```bash
# Verify certificate matches FQDN
openssl s_client -connect dc01.corp.example.com:636 < /dev/null 2>/dev/null \
  | openssl x509 -noout -text \
  | grep -E "Subject:|DNS:|IP:"

# Expected output for a well-configured AD cert:
# Subject: CN=dc01.corp.example.com
# DNS:dc01.corp.example.com
# DNS:dc01
# DNS:corp.example.com
```

## Docker Compose with Extra Hosts

```yaml
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    extra_hosts:
      # Ensure FQDN resolves inside the container
      - "dc01.corp.example.com:192.168.1.10"
      - "dc02.corp.example.com:192.168.1.11"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    command:
      - "--trusted-origins=https://portainer.example.com"

volumes:
  portainer_data:
```

## Conclusion

Use FQDN for Active Directory connections in Portainer — it's required for proper TLS certificate validation and follows AD best practices. If DNS resolution is unreliable in your Docker network, use `extra_hosts` in Docker Compose to add the domain controller's hostname manually. Only fall back to IP addresses when absolutely necessary, and disable certificate verification only as a last resort in non-production environments.
