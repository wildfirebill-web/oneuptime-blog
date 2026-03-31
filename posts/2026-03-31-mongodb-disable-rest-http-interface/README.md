# How to Disable MongoDB REST Interface and HTTP Interface

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Security, HTTP, REST, Hardening

Description: Learn how to disable MongoDB's legacy REST and HTTP interfaces to reduce your attack surface and meet security hardening requirements.

---

## Overview

Older MongoDB versions (before 3.6) shipped with an HTTP status interface on port 28017 and an optional REST interface. These interfaces expose operational information and in some configurations allow unauthenticated read access. Even in newer versions where these are disabled by default, explicitly configuring them as off is an important security hardening step.

## Checking If the HTTP Interface Is Running

```bash
# Check if port 28017 is open
curl http://localhost:28017/

# Or use netstat
netstat -tulpn | grep 28017
```

If you receive an HTML page or JSON response, the HTTP interface is active.

## Disabling in mongod.conf (Recommended)

The authoritative way to disable these interfaces is through the configuration file:

```yaml
# /etc/mongod.conf
net:
  port: 27017
  bindIp: 127.0.0.1

# MongoDB 3.2 and below: explicitly disable HTTP
# net:
#   http:
#     enabled: false
#     RESTInterfaceEnabled: false

security:
  authorization: enabled
```

On MongoDB 3.6+, the HTTP interface is removed entirely and these settings are no longer needed. However, for older deployments still running MongoDB 3.4 or earlier, these settings must be explicitly set.

## Disabling via Command-Line Flags

For older MongoDB versions started from the command line:

```bash
mongod \
  --port 27017 \
  --dbpath /var/lib/mongodb \
  --nohttpinterface \
  --norest \
  --auth
```

The `--nohttpinterface` flag disables the HTTP status page and the `--norest` flag disables the REST API.

## Blocking Port 28017 at the Firewall Level

Even if the interface is disabled in configuration, add a firewall rule as defense-in-depth:

```bash
# Using ufw (Ubuntu)
sudo ufw deny 28017

# Using iptables
sudo iptables -A INPUT -p tcp --dport 28017 -j DROP

# Using firewalld (RHEL/CentOS)
sudo firewall-cmd --permanent --remove-port=28017/tcp
sudo firewall-cmd --reload
```

## Verifying MongoDB Is Not Exposing Administrative Ports

Run a port scan from another host to verify no administrative ports are accessible:

```bash
nmap -p 27017,28017 your-mongodb-host
```

Both ports should show as filtered or closed from external hosts. Port 27017 should be accessible only from trusted application servers, never from the public internet.

## Additional MongoDB Network Hardening

```yaml
# mongod.conf - bind only to specific IPs
net:
  port: 27017
  bindIp: 10.0.0.5,127.0.0.1  # Only private network + localhost

security:
  authorization: enabled

# Disable server-side JavaScript execution if not needed
security:
  javascriptEnabled: false
```

## Verifying Configuration Is Applied

```javascript
// Check current server parameters
db.adminCommand({ getCmdLineOpts: 1 })
```

Review the `parsed` section of the output to confirm your configuration file settings are loaded correctly.

## MongoDB Version Guidance

| MongoDB Version | HTTP Interface Status |
|----------------|----------------------|
| 2.x, 3.0-3.4   | Enabled by default - must explicitly disable |
| 3.6+           | Removed entirely - no action needed |

If you are on MongoDB 3.6 or later, the HTTP and REST interfaces do not exist and no configuration is needed. Focus hardening efforts on disabling unused authentication mechanisms and binding to private IPs.

## Summary

MongoDB's legacy HTTP and REST interfaces expose operational data and pose a security risk when left enabled. Disable them in `mongod.conf` using `net.http.enabled: false` and `net.http.RESTInterfaceEnabled: false` for MongoDB versions 3.4 and earlier. Block port 28017 at the firewall as defense-in-depth. MongoDB 3.6+ removes these interfaces entirely. Combine this with `bindIp` restrictions, `authorization: enabled`, and port 27017 firewall rules for a properly hardened deployment.
