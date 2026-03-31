# How to Run MongoDB Behind a Firewall

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Security, Firewall, Network, Operations

Description: Configure firewall rules to protect MongoDB from unauthorized access while allowing replica set replication and application connections.

---

## Ports MongoDB Uses

Before configuring firewall rules, understand which ports MongoDB uses:

- `27017` - Default mongod and mongos port
- `27018` - Default when using `--shardsvr`
- `27019` - Default when using `--configsvr`
- Custom ports if configured differently

## Basic iptables Rules for a Standalone MongoDB

Allow application servers to connect; block everything else:

```bash
# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH (don't lock yourself out)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow MongoDB from application servers only
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.20 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.21 -j ACCEPT

# Drop all other MongoDB connections
iptables -A INPUT -p tcp --dport 27017 -j DROP

# Save rules
iptables-save > /etc/iptables/rules.v4
```

## Replica Set Firewall Rules

Replica set members must be able to connect to each other on port 27017 (or your custom port). Each member needs both inbound and outbound rules.

```bash
# On each replica set member, allow all replica set peers

# Member 1 (this server): 10.0.1.10
# Member 2: 10.0.1.11
# Member 3: 10.0.1.12
# App servers: 10.0.1.20, 10.0.1.21

# Allow from other replica set members
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.11 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.12 -j ACCEPT

# Allow from application servers
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.20 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.21 -j ACCEPT

# Drop all else
iptables -A INPUT -p tcp --dport 27017 -j DROP
```

## Sharded Cluster Firewall Rules

In a sharded cluster, mongos routers connect to config servers and shards. Config servers and shards only need to be reachable from mongos and other cluster members.

```bash
# On config servers - allow from mongos only
iptables -A INPUT -p tcp --dport 27019 -s 10.0.2.10 -j ACCEPT  # mongos
iptables -A INPUT -p tcp --dport 27019 -s 10.0.2.11 -j ACCEPT  # config peer 2
iptables -A INPUT -p tcp --dport 27019 -j DROP

# On shards - allow from mongos and replica peers
iptables -A INPUT -p tcp --dport 27018 -s 10.0.2.10 -j ACCEPT  # mongos
iptables -A INPUT -p tcp --dport 27018 -s 10.0.3.11 -j ACCEPT  # shard peer
iptables -A INPUT -p tcp --dport 27018 -j DROP

# App servers connect only to mongos (port 27017)
```

## Using UFW (Ubuntu)

```bash
# Reset and configure UFW
ufw default deny incoming
ufw allow ssh

# Allow MongoDB from application CIDR
ufw allow from 10.0.1.0/24 to any port 27017
ufw allow from 10.0.1.0/24 to any port 27018  # shards
ufw allow from 10.0.1.0/24 to any port 27019  # config servers

ufw enable
```

## Testing the Firewall

From an authorized host (should succeed):

```bash
nc -zv 10.0.1.10 27017  # Should print "Connection to ... succeeded"
```

From an unauthorized host (should fail or timeout):

```bash
nc -zv -w 3 10.0.1.10 27017  # Should print "Connection refused" or timeout
```

## Summary

Running MongoDB behind a firewall requires allowing traffic on port 27017 (and 27018/27019 for sharded clusters) only from known application servers and replica set/shard peers. Use iptables or UFW rules to whitelist specific source IPs and drop all other connections to MongoDB ports. Always test rules from both authorized and unauthorized hosts before finalizing.
