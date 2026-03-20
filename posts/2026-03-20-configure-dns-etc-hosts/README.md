# How to Configure Custom DNS Entries in /etc/hosts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: DNS, /etc/hosts, Linux, Networking, Configuration, Override

Description: Use /etc/hosts to create local DNS overrides, override specific hostnames, and manage local development DNS without modifying a DNS server.

## Introduction

`/etc/hosts` is the simplest DNS override mechanism. Entries here are resolved before DNS is queried (by default, per `/etc/nsswitch.conf`). It is useful for overriding specific hostnames for testing, creating local development aliases, pointing services to different servers for testing, or working around DNS issues with specific hosts.

## Basic /etc/hosts Format

```
# Format:
# IP_ADDRESS  HOSTNAME  [ALIAS...]

# Standard entries:
127.0.0.1   localhost
127.0.1.1   myhostname.local myhostname
::1         localhost ip6-localhost ip6-loopback

# Custom entries:
10.20.0.10  api.example.com
10.20.0.11  db.example.com db
192.168.1.100  dev.local

# Multiple aliases:
10.20.0.10  api.example.com api-server api
```

## Common Use Cases

```bash
# 1. Local development: point domain to localhost
echo "127.0.0.1 myapp.local" >> /etc/hosts
# Test your app via http://myapp.local

# 2. Override specific host during testing:
echo "10.20.0.99 api.example.com" >> /etc/hosts
# All connections to api.example.com go to test server

# 3. Block domains (point to 127.0.0.1 or 0.0.0.0):
echo "0.0.0.0 tracking.example.com" >> /etc/hosts
# Blocks tracking.example.com (resolves to loopback, connection refused)

# 4. Speed up repeated lookups (override with known IP):
echo "1.2.3.4 api.external-service.com" >> /etc/hosts
# Bypasses DNS entirely for this host

# 5. Container-to-container resolution in Docker:
# Docker usually handles this via DNS, but for custom overrides:
docker run --add-host="myservice:10.20.0.5" myimage
# Adds to /etc/hosts inside container
```

## Manage /etc/hosts Safely

```bash
# Always backup before editing:
cp /etc/hosts /etc/hosts.backup

# Edit with your preferred editor:
nano /etc/hosts
# or: vim /etc/hosts

# Add multiple entries at once:
cat >> /etc/hosts << 'EOF'
# Development servers
10.20.0.10  api.example.com
10.20.0.11  db.example.com
10.20.0.12  cache.example.com
EOF

# Remove an entry (using sed):
sed -i '/api.example.com/d' /etc/hosts

# Verify the entry is active:
getent hosts api.example.com
# Should return the IP you configured

# Or:
ping -c 1 api.example.com
# Check that it resolves to expected IP
```

## /etc/hosts in nsswitch.conf Order

```bash
# Check the resolution order:
grep ^hosts /etc/nsswitch.conf
# Standard: hosts: files dns
# "files" = /etc/hosts, checked FIRST before DNS

# If "dns" comes before "files":
# hosts: dns files
# /etc/hosts is only checked if DNS fails (unusual)

# Override order for testing (without changing nsswitch.conf):
getent -s files hosts api.example.com   # Only check /etc/hosts
getent -s dns hosts api.example.com     # Only check DNS
```

## Docker and Container /etc/hosts

```bash
# Docker automatically adds entries to container /etc/hosts:
# - Container's own hostname
# - Other containers in the same network (by container name)

# Inspect a running container's /etc/hosts:
docker exec mycontainer cat /etc/hosts

# Add custom hosts when running a container:
docker run --add-host="prod-db:10.20.0.20" myapp
docker run --add-host="legacy-api:192.168.1.50" myapp

# Docker Compose: add extra_hosts:
# services:
#   app:
#     extra_hosts:
#       - "prod-db:10.20.0.20"
#       - "legacy-api:192.168.1.50"
```

## Automation and Infrastructure

```bash
# Use Ansible to manage /etc/hosts entries:
# tasks:
#   - name: Add custom DNS entries
#     lineinfile:
#       path: /etc/hosts
#       line: "{{ item.ip }} {{ item.hostname }}"
#       state: present
#     loop:
#       - { ip: '10.20.0.10', hostname: 'api.internal.example.com' }
#       - { ip: '10.20.0.11', hostname: 'db.internal.example.com' }

# Check for conflicting entries:
awk '/^[^#]/{ip=$1; for(i=2;i<=NF;i++) print $i, ip}' /etc/hosts | \
  awk '{seen[$1]++; if(seen[$1]>1) print "DUPLICATE:", $1}'
```

## Conclusion

`/etc/hosts` provides immediate, no-DNS-server-required hostname resolution. It is resolved before DNS by default (`files` before `dns` in `/etc/nsswitch.conf`). Use it for local development overrides, host-specific testing, and temporary routing changes. For blocking domains, point them to `0.0.0.0` (faster failure than `127.0.0.1` which sends the connection to your localhost). Remember that `/etc/hosts` changes take effect immediately — no caching or TTL — but only affect the single host where the file exists.
