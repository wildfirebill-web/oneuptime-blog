# How to Configure MongoDB for Dual-Stack (IPv4 and IPv6)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, IPv4, IPv6, Dual-Stack, Network

Description: Learn how to configure MongoDB to listen on both IPv4 and IPv6 addresses simultaneously for dual-stack network environments.

---

## What Is Dual-Stack?

A dual-stack configuration allows MongoDB to accept connections over both IPv4 and IPv6 simultaneously. This is the right choice during a migration from IPv4 to IPv6, in environments where clients use a mix of protocols, or in cloud networks that allocate both address types.

## Configuring Dual-Stack in mongod.conf

Enable IPv6 and include both IPv4 and IPv6 bind addresses:

```yaml
net:
  port: 27017
  bindIp: "127.0.0.1,::1,10.0.1.5,2001:db8::5"
  ipv6: true
```

This binds to:
- `127.0.0.1` - IPv4 loopback
- `::1` - IPv6 loopback
- `10.0.1.5` - IPv4 network interface
- `2001:db8::5` - IPv6 network interface

After editing, restart mongod:

```bash
sudo systemctl restart mongod
```

## Using bindIpAll for Dual-Stack

`bindIpAll: true` combined with `ipv6: true` binds to all available IPv4 and IPv6 interfaces:

```yaml
net:
  port: 27017
  bindIpAll: true
  ipv6: true
```

This is convenient for development but use specific addresses in production.

## Verifying Dual-Stack Binding

```bash
ss -tlnp | grep mongod
```

Expected output showing both IPv4 and IPv6:

```text
State    Recv-Q  Send-Q  Local Address:Port    Peer Address:Port
LISTEN   0       128     127.0.0.1:27017       0.0.0.0:*
LISTEN   0       128     10.0.1.5:27017        0.0.0.0:*
LISTEN   0       128     [::1]:27017           [::]:*
LISTEN   0       128     [2001:db8::5]:27017   [::]:*
```

## Connecting with Both Protocol Types

Clients can use either protocol:

```javascript
// IPv4 connection
const client4 = new MongoClient("mongodb://127.0.0.1:27017");

// IPv6 connection
const client6 = new MongoClient("mongodb://[::1]:27017");

// Using a hostname (client resolves which protocol to use)
const clientHost = new MongoClient("mongodb://mongo.internal:27017");
```

For replica sets, use hostnames rather than addresses so clients can resolve using whichever protocol is available.

## Replica Set Configuration for Dual-Stack

When all nodes have both IPv4 and IPv6 addresses, use fully qualified domain names in the replica set configuration:

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "node1.internal:27017" },
    { _id: 1, host: "node2.internal:27017" },
    { _id: 2, host: "node3.internal:27017" }
  ]
});
```

Ensure your DNS returns both A (IPv4) and AAAA (IPv6) records for each hostname.

## Firewall Rules for Dual-Stack

Apply firewall rules to both protocol stacks:

```bash
# IPv4 rules (iptables)
sudo iptables -A INPUT -p tcp --dport 27017 -s 10.0.0.0/8 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 27017 -j DROP

# IPv6 rules (ip6tables)
sudo ip6tables -A INPUT -p tcp --dport 27017 -s 2001:db8::/32 -j ACCEPT
sudo ip6tables -A INPUT -p tcp --dport 27017 -j DROP
```

Forgetting IPv6 firewall rules when adding IPv4 rules is a common security oversight.

## Summary

Enable dual-stack MongoDB by setting `ipv6: true` and listing both IPv4 and IPv6 addresses in `net.bindIp`. Use `bindIpAll: true` only in development. Verify both protocol families appear in `ss -tlnp` output. Use hostnames in replica set configurations and apply firewall rules to both iptables and ip6tables. Dual-stack operation is transparent to most clients - they connect using whichever protocol their DNS resolution returns.
