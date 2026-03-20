# How to Set Up MongoDB Authentication with IPv4 Access Control

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Authentication, IPv4, Security, Access Control, Configuration, Database

Description: Learn how to enable MongoDB authentication and configure IPv4-based access control to secure your MongoDB instance from unauthorized connections.

---

By default MongoDB runs without authentication on localhost. For production deployments, enable authentication and restrict network access to specific IPv4 addresses.

## Step 1: Bind MongoDB to a Specific IPv4 Address

```yaml
# /etc/mongod.conf

net:
  port: 27017
  # Bind to localhost and the server's internal IPv4 only
  # Never bind to 0.0.0.0 in production without a firewall
  bindIp: 127.0.0.1,10.0.0.10

security:
  authorization: disabled   # Disabled initially to create the first admin user
```

```bash
systemctl restart mongod
```

## Step 2: Create the Admin User (Before Enabling Auth)

```javascript
// Connect to MongoDB while auth is disabled
mongosh --host 127.0.0.1 --port 27017

// Switch to the admin database
use admin

// Create the site administrator
db.createUser({
  user: "mongoadmin",
  pwd: "StrongAdminPassword123!",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})

// Create a read/write user for a specific database
use myapp
db.createUser({
  user: "appuser",
  pwd: "AppPassword456!",
  roles: [{ role: "readWrite", db: "myapp" }]
})
```

## Step 3: Enable Authentication

```yaml
# /etc/mongod.conf

net:
  port: 27017
  bindIp: 127.0.0.1,10.0.0.10

security:
  authorization: enabled    # Enable authentication
```

```bash
systemctl restart mongod
```

## Step 4: Restrict Access with a Firewall

```bash
# Allow MongoDB connections only from the application server's IPv4
ufw allow from 10.0.0.20 to any port 27017
ufw allow from 127.0.0.1 to any port 27017
ufw deny 27017
# or
iptables -A INPUT -p tcp --dport 27017 -s 10.0.0.20 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -s 127.0.0.1 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -j DROP
```

## Connecting with Authentication

```bash
# Connect as the app user
mongosh "mongodb://appuser:AppPassword456!@10.0.0.10:27017/myapp"

# Or with mongosh
mongosh --host 10.0.0.10 --port 27017 -u appuser -p AppPassword456! --authenticationDatabase myapp
```

## Creating a Read-Only User

```javascript
use reporting
db.createUser({
  user: "reporter",
  pwd: "ReadOnlyPass789!",
  roles: [{ role: "read", db: "myapp" }]
})
```

## Verifying Authentication is Active

```bash
# This should fail with "command find requires authentication"
mongosh --host 10.0.0.10 --port 27017 --eval "db.getSiblingDB('myapp').myCollection.find()"

# This should succeed
mongosh "mongodb://appuser:AppPassword456!@10.0.0.10:27017/myapp" --eval "db.myCollection.findOne()"
```

## Key Takeaways

- Create the admin user before enabling authentication, or you'll lock yourself out.
- `bindIp` in `mongod.conf` restricts which IPv4 addresses MongoDB listens on.
- Always combine `bindIp` restrictions with firewall rules for defense in depth.
- Use role-based access control (RBAC) to give each application user only the minimum required permissions.
