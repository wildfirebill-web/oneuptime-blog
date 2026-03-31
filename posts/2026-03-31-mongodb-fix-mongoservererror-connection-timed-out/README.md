# How to Fix MongoServerSelectionError: Connection Timed Out

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Connection, Error, Troubleshooting, Network

Description: Diagnose and fix MongoServerSelectionError connection timeout errors in MongoDB by checking network access, authentication, connection strings, and driver settings.

---

## What Is MongoServerSelectionError?

`MongoServerSelectionError: connection timed out` occurs when the MongoDB driver cannot find a suitable server to handle an operation within the `serverSelectionTimeoutMS` window (default: 30 seconds).

This error does not necessarily mean MongoDB is down - it means the driver could not reach or select a server. Common causes:

- MongoDB is not running or is unreachable due to a network rule.
- The IP address or port is wrong in the connection string.
- A firewall or security group is blocking the connection.
- TLS/authentication misconfiguration.
- DNS resolution failure.

## Step 1 - Verify MongoDB Is Running

On the server hosting MongoDB:

```bash
sudo systemctl status mongod
```

If the service is not running, start it:

```bash
sudo systemctl start mongod
```

Check the MongoDB log for startup errors:

```bash
sudo journalctl -u mongod --since "5 minutes ago"
```

## Step 2 - Test Network Connectivity

From your application server, test that the MongoDB port (default 27017) is reachable:

```bash
nc -zv mongodb-host 27017
```

Or with telnet:

```bash
telnet mongodb-host 27017
```

If the connection is refused or times out, the issue is network-level - check firewalls, security groups, and VPC routing rules.

## Step 3 - Verify the Connection String

A typical connection string looks like:

```
mongodb://username:password@host:27017/dbname?authSource=admin
```

Common mistakes:
- Wrong port (e.g., `27018` instead of `27017`).
- Missing `authSource` parameter when using admin credentials.
- Incorrectly URL-encoded special characters in the password.

For replica sets, ensure all members are listed:

```
mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=myRS
```

## Step 4 - Check MongoDB Bind IP

By default, MongoDB only listens on `127.0.0.1`. If you want it to accept remote connections, edit `/etc/mongod.conf`:

```yaml
net:
  bindIp: 0.0.0.0
  port: 27017
```

Then restart:

```bash
sudo systemctl restart mongod
```

Never expose MongoDB to `0.0.0.0` without a firewall restricting access to trusted IPs.

## Step 5 - Increase serverSelectionTimeoutMS

If MongoDB is reachable but slow to respond (e.g., under high load), you may hit the default 30-second timeout. Increase it in your driver configuration:

```javascript
const client = new MongoClient(uri, {
  serverSelectionTimeoutMS: 60000,  // 60 seconds
  connectTimeoutMS: 30000
});
```

## Step 6 - Check TLS/SSL Configuration

If TLS is required, ensure the CA certificate is trusted and the client is configured to use it:

```javascript
const client = new MongoClient(uri, {
  tls: true,
  tlsCAFile: "/path/to/ca.pem"
});
```

A TLS handshake failure can also manifest as a connection timeout.

## Summary

`MongoServerSelectionError: connection timed out` is a connectivity problem, not a MongoDB bug. Work through the checklist: verify the service is running, test network reachability on port 27017, confirm the connection string and bind IP settings, and review firewall rules. Once the underlying network or configuration issue is resolved, the driver will connect successfully.
