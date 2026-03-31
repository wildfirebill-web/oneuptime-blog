# How to Change the Default MongoDB Port

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Port, Network, Configuration, Security

Description: Learn how to change MongoDB's default port from 27017 to a custom port and update all clients, replica set configurations, and firewall rules accordingly.

---

## Why Change the Default Port?

MongoDB's default port `27017` is well-known and a common target for automated scans. Changing it provides minimal security through obscurity (never a substitute for authentication and firewall rules), but also:

- Allows running multiple MongoDB instances on the same host
- Satisfies organizational port assignment policies
- Separates development and production instances on the same server

## Changing the Port in mongod.conf

Edit `/etc/mongod.conf`:

```yaml
net:
  port: 27117
  bindIp: 127.0.0.1
```

Restart mongod:

```bash
sudo systemctl restart mongod
```

Verify the new port is listening:

```bash
ss -tlnp | grep mongod
# Should show :27117
```

## Changing the Port via Command Line

```bash
mongod --port 27117 --dbpath /var/lib/mongodb --logpath /var/log/mongodb/mongod.log
```

## Connecting to a Non-Default Port

Update all client connection strings to include the custom port:

```javascript
// Node.js
const client = new MongoClient("mongodb://localhost:27117");

// Python
from pymongo import MongoClient
client = MongoClient("mongodb://localhost:27117/")
```

```bash
# mongosh
mongosh "mongodb://localhost:27117"

# mongodump/mongorestore
mongodump --port 27117 --db mydb --out /backup/
mongorestore --port 27117 --db mydb /backup/mydb/
```

## Updating Replica Set Members

If you change the port on a running replica set, update the replica set configuration after restarting each member on the new port:

```javascript
// Connect to the primary on the new port
const cfg = rs.conf();
cfg.members[0].host = "node1.internal:27117";
rs.reconfig(cfg, { force: true });
```

Change one member at a time to maintain availability.

## Running Multiple Instances on the Same Host

Use different ports for each instance:

```bash
# Instance 1: production
mongod --port 27017 --dbpath /data/prod --fork --logpath /var/log/mongod-prod.log

# Instance 2: staging
mongod --port 27117 --dbpath /data/staging --fork --logpath /var/log/mongod-staging.log
```

Or use separate `mongod.conf` files and systemd service units for each instance.

## Updating Firewall Rules

After changing the port, update your firewall rules:

```bash
# Remove old rule
sudo iptables -D INPUT -p tcp --dport 27017 -j ACCEPT

# Add new rule
sudo iptables -A INPUT -p tcp --dport 27117 -s 10.0.0.0/8 -j ACCEPT

# Save rules
sudo iptables-save > /etc/iptables/rules.v4
```

For UFW:

```bash
sudo ufw allow from 10.0.0.0/8 to any port 27117 proto tcp
sudo ufw delete allow 27017
```

## Summary

Change MongoDB's port by editing `net.port` in `mongod.conf` and restarting mongod. Update all connection strings, mongodump/mongorestore invocations, monitoring agents, and replica set host configurations to use the new port. Update firewall rules for both old and new ports. Port changes are straightforward for standalone instances but require a rolling reconfiguration for replica sets to maintain availability.
