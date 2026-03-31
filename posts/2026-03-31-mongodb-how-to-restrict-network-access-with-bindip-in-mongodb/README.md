# How to Restrict Network Access with bindIp in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Security, Network, bindIp, Configuration

Description: Use MongoDB bindIp configuration to control which network interfaces mongod listens on and prevent connections from untrusted network addresses.

---

## Understanding bindIp

The `bindIp` setting in `mongod.conf` tells MongoDB which network interface IP addresses to accept connections on. By default since MongoDB 3.6, `bindIp` is set to `127.0.0.1`, meaning MongoDB only accepts localhost connections. This is a security improvement over older versions that bound to all interfaces by default.

## Checking the Current bindIp

```javascript
// Connect to mongod and check
db.adminCommand({ getCmdLineOpts: 1 }).parsed.net.bindIp
```

Or check the config file:

```bash
grep -A2 "^net:" /etc/mongod.conf
```

## Binding to a Single Private IP

Bind MongoDB to only the private network interface used by application servers:

```yaml
# /etc/mongod.conf
net:
  port: 27017
  bindIp: "127.0.0.1,10.0.1.5"
```

After this change, `mongod` listens on both `127.0.0.1:27017` (for local tools) and `10.0.1.5:27017` (for applications on the 10.0.1.x network). Connections from public IPs or other networks are refused at the OS level.

Apply the change:

```bash
sudo systemctl restart mongod
```

## Binding to All Interfaces (Not Recommended for Production)

Some deployments bind to all interfaces and rely solely on authentication and firewall rules. Avoid this when possible:

```yaml
net:
  bindIp: "0.0.0.0"  # All IPv4 interfaces
```

If IPv6 is also needed:

```yaml
net:
  bindIp: "0.0.0.0,::"  # All IPv4 and IPv6
```

## Verifying the Binding

Check what ports mongod is listening on after restart:

```bash
ss -tlnp | grep mongod
# or
netstat -tlnp | grep 27017
```

You should see only the interfaces you specified in bindIp.

## bindIp for Replica Set Members

Each replica set member must bind to an IP that other members can reach. If a member only binds to `127.0.0.1`, other members cannot connect.

```yaml
# On each replica set node - bind to localhost and the private IP
net:
  bindIp: "127.0.0.1,192.168.1.10"
```

Ensure each member's replica set `host` entry matches a bound IP:

```javascript
rs.conf().members.map(m => m.host)
// Should list IPs that each member is actually binding to
```

## bindIp vs Firewall Rules

`bindIp` and firewall rules serve different purposes. `bindIp` controls which interfaces MongoDB listens on (before any packet is accepted). Firewall rules control which source IPs can reach the port. Use both together:

1. Set `bindIp` to your private/internal interface
2. Use iptables or a security group to whitelist only known application server IPs

```bash
# After setting bindIp to 10.0.1.5, add a firewall rule
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -j DROP
```

## Summary

`bindIp` is MongoDB's first line of network defense, controlling which interfaces it listens on. Always set it to specific private IP addresses rather than `0.0.0.0` in production. Combine with firewall rules to whitelist application server source IPs. For replica sets, ensure each member binds to the IP listed in the replica set configuration so members can replicate from each other.
