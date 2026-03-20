# How to Configure MongoDB bindIp for Specific IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, IPv4, BindIp, Configuration, Security, Database

Description: Configure MongoDB's bindIp setting in mongod.conf to listen on specific IPv4 addresses, restrict network exposure, and verify the binding is correct.

## Introduction

MongoDB defaults to binding to `127.0.0.1` (localhost only). To accept remote connections, add your server's IPv4 address to `bindIp`. For multi-homed servers, specifying the exact interface prevents MongoDB from being accessible on untrusted interfaces.

## Configuration

```yaml
# /etc/mongod.conf

net:
  port: 27017
  # Bind to specific IPv4 (comma-separated, no spaces)
  bindIp: 127.0.0.1,10.0.0.5

  # Or bind to all IPv4 interfaces (use with firewall)
  # bindIp: 0.0.0.0

  # Disable IPv6
  ipv6: false
```

```bash
# Apply changes

sudo systemctl restart mongod

# Verify MongoDB is listening on expected addresses
sudo ss -tlnp | grep mongod
# Expected: 127.0.0.1:27017 and 10.0.0.5:27017
```

## Using Environment Variable

```bash
# mongod.conf supports YAML anchors but not environment variable substitution
# Pass bindIp via command line instead:
mongod --bind_ip 127.0.0.1,10.0.0.5 --config /etc/mongod.conf

# Or in the systemd service override:
sudo systemctl edit mongod
# [Service]
# ExecStart=
# ExecStart=/usr/bin/mongod --config /etc/mongod.conf --bind_ip 127.0.0.1,10.0.0.5
```

## Enabling Authentication (Required for Remote Access)

```bash
# /etc/mongod.conf
security:
  authorization: enabled

# Create admin user (connect locally first without auth):
mongosh

use admin
db.createUser({
  user: "admin",
  pwd: "AdminPassword123",
  roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
})

# Create application user
use appdb
db.createUser({
  user: "appuser",
  pwd: "AppPassword123",
  roles: [ { role: "readWrite", db: "appdb" } ]
})
```

## Firewall Rules

```bash
# Allow MongoDB from specific IPs only
sudo ufw allow from 10.0.0.10 to any port 27017
sudo ufw allow from 10.0.0.11 to any port 27017
sudo ufw deny 27017

# iptables
sudo iptables -A INPUT -p tcp --dport 27017 -s 10.0.0.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 27017 -j DROP
```

## Testing Remote Connection

```bash
# From a remote client:
mongosh "mongodb://appuser:AppPassword123@10.0.0.5:27017/appdb"

# Test connectivity
nc -zv 10.0.0.5 27017
# Expected: Connection to 10.0.0.5 27017 port succeeded

# Check MongoDB logs
sudo tail -30 /var/log/mongodb/mongod.log | grep -E "bind|listen|port"
```

## Conclusion

Set MongoDB `bindIp` to include both `127.0.0.1` and your server's IPv4 address for remote access. Enable `security.authorization` whenever accepting remote connections. Use firewall rules to limit which client IPs can reach port 27017. Never expose MongoDB on `0.0.0.0` without authentication and firewall restrictions.
