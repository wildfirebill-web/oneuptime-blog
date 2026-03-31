# How to Fix MongoNetworkError: Connect ECONNREFUSED in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Network Error, Econnrefused, Troubleshooting, Connection

Description: Diagnose and fix the MongoNetworkError Connect ECONNREFUSED error in MongoDB by checking the server status, port, firewall, and connection string configuration.

---

## Understanding the Error

`MongoNetworkError: connect ECONNREFUSED 127.0.0.1:27017` means the TCP connection to MongoDB was actively refused. This happens when no process is listening on the target host and port.

```text
MongoNetworkError: connect ECONNREFUSED 127.0.0.1:27017
    at connectionFailureError (node_modules/mongodb/lib/core/connection/connect.js)
```

## Step 1: Check If MongoDB Is Running

```bash
# systemd
sudo systemctl status mongod

# macOS (Homebrew)
brew services list | grep mongodb

# check if the process is listening
sudo ss -tlnp | grep 27017
# or on macOS
lsof -i :27017
```

If the service is stopped, start it:

```bash
sudo systemctl start mongod
# or
brew services start mongodb-community
```

## Step 2: Check the MongoDB Log for Startup Errors

If the service fails to start, inspect the log:

```bash
sudo journalctl -u mongod -n 50 --no-pager
# or
tail -100 /var/log/mongodb/mongod.log
```

Common startup failures include: corrupted data files, wrong permissions on `/var/lib/mongodb`, or a port already in use.

## Step 3: Verify the Correct Host and Port

MongoDB may be bound to a different address than expected. Check `mongod.conf`:

```bash
grep -E "bindIp|port" /etc/mongod.conf
```

Example `mongod.conf` section:

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1
```

If you are connecting from a remote host, change `bindIp` to `0.0.0.0` (and secure with firewall rules):

```yaml
net:
  port: 27017
  bindIp: 0.0.0.0
```

Restart after changes:

```bash
sudo systemctl restart mongod
```

## Step 4: Check Your Connection String

Make sure the host and port in your application match what MongoDB is actually listening on:

```javascript
// Node.js
const client = new MongoClient('mongodb://localhost:27017');
```

```python
# Python
client = MongoClient('mongodb://localhost:27017/')
```

If MongoDB runs in Docker, use the container's published port and the host machine's IP (or `host.docker.internal` on Docker Desktop):

```bash
docker run -d -p 27017:27017 --name mongo mongo:7
```

```javascript
const client = new MongoClient('mongodb://host.docker.internal:27017');
```

## Step 5: Check Firewall Rules

If connecting across a network, ensure the port is open:

```bash
# Ubuntu/Debian
sudo ufw allow 27017/tcp

# RHEL/CentOS
sudo firewall-cmd --permanent --add-port=27017/tcp
sudo firewall-cmd --reload
```

For AWS security groups, verify inbound rules allow TCP on port 27017 from your application's IP range.

## Step 6: Replica Set or Atlas Issues

If using a replica set URI, the driver will try all hosts in the seed list. Ensure all listed hosts are reachable:

```text
mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=rs0
```

For Atlas, whitelist your IP address in the Network Access settings and use the Atlas-provided SRV connection string.

## Summary

`ECONNREFUSED` means nothing is listening on the target port. Start with the basics - confirm mongod is running, check the bound IP and port in `mongod.conf`, verify your connection string matches, and check firewall rules. For containerized or cloud environments, ensure published ports and network policies allow the connection.
