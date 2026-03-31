# How to Create a Pod with Custom DNS in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Pod, DNS, Networking

Description: Learn how to configure custom DNS servers and search domains for Podman pods.

---

> Custom DNS settings let you control how containers in a pod resolve domain names.

By default, containers inherit DNS configuration from the host. In many environments, you need to point containers at specific DNS servers for internal service discovery, split-horizon DNS, or to use a custom resolver. Podman lets you set DNS servers, search domains, and options at the pod level.

---

## Creating a Pod with Custom DNS Servers

```bash
# Create a pod that uses Cloudflare and Google DNS

podman pod create --name dns-pod \
  --dns 1.1.1.1 \
  --dns 8.8.8.8

# Verify DNS configuration inside a container
podman run --rm --pod dns-pod docker.io/library/alpine cat /etc/resolv.conf
# nameserver 1.1.1.1
# nameserver 8.8.8.8
```

## Setting DNS Search Domains

```bash
# Create a pod with custom search domains
podman pod create --name search-pod \
  --dns 10.0.0.53 \
  --dns-search example.com \
  --dns-search internal.corp

# Verify the search domains
podman run --rm --pod search-pod docker.io/library/alpine cat /etc/resolv.conf
# nameserver 10.0.0.53
# search example.com internal.corp
```

## Setting DNS Options

```bash
# Create a pod with DNS options
podman pod create --name opt-pod \
  --dns 10.0.0.53 \
  --dns-option ndots:5 \
  --dns-option timeout:2

# Check the options
podman run --rm --pod opt-pod docker.io/library/alpine cat /etc/resolv.conf
# options ndots:5 timeout:2
```

## Use Case: Internal Service Discovery

```bash
# Point pods at an internal DNS server for service discovery
podman pod create --name internal-pod \
  --dns 10.10.0.2 \
  --dns-search svc.cluster.local

# Containers can resolve internal services by short name
podman run --rm --pod internal-pod docker.io/library/alpine \
  nslookup api-server
# Resolves api-server.svc.cluster.local via 10.10.0.2
```

## Use Case: Air-Gapped Environment

```bash
# In an air-gapped environment, use a local DNS server only
podman pod create --name airgap-pod \
  --dns 192.168.1.1 \
  --dns-search local.lan

podman run -d --pod airgap-pod --name app docker.io/library/alpine sleep 3600

# Verify DNS resolution works with the local server
podman exec app nslookup registry.local.lan
```

## Verifying DNS Resolution

```bash
# Test DNS resolution from within the pod
podman run --rm --pod dns-pod docker.io/library/alpine \
  sh -c "apk add --no-cache bind-tools && dig example.com +short"

# Test with nslookup
podman run --rm --pod dns-pod docker.io/library/alpine \
  nslookup example.com
```

## Summary

Configure custom DNS for Podman pods using `--dns` for nameservers, `--dns-search` for search domains, and `--dns-option` for resolver options. All containers in the pod inherit these settings. This is essential for internal service discovery, split-horizon DNS, and air-gapped environments.
