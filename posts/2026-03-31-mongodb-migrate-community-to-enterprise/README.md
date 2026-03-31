# How to Migrate from MongoDB Community to MongoDB Enterprise

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Enterprise, Migration, Security, Operations

Description: Learn how to upgrade from MongoDB Community Edition to MongoDB Enterprise, including installation, security features, auditing, and LDAP configuration.

---

## Overview

MongoDB Enterprise Advanced includes features not available in Community Edition: auditing, LDAP authentication, Kerberos support, encrypted storage, SNMP monitoring, and MongoDB Ops Manager. Migrating from Community to Enterprise is an in-place upgrade - your data does not move.

## Key Enterprise Features

Before migrating, understand what you gain with Enterprise:

```text
MongoDB Enterprise Advanced features:
- Audit logging: track who did what, when
- LDAP/Active Directory authentication and authorization
- Kerberos authentication
- Encrypted storage (WiredTiger Encrypted At Rest)
- SNMP monitoring integration
- MongoDB Ops Manager (on-prem cluster management)
- In-memory storage engine
- Queryable Encryption (CSFLE)
```

## Verify Your Current Installation

```bash
# Check current MongoDB version and edition
mongod --version
# Output: db version v7.0.x (Community)

# Confirm no Enterprise flag
mongod --version | grep -i enterprise
# No output = Community Edition
```

## Install MongoDB Enterprise

On Ubuntu/Debian, add the Enterprise repository and install:

```bash
# Add MongoDB Enterprise repository (Ubuntu 22.04)
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.com/apt/ubuntu jammy/mongodb-enterprise/7.0 multiverse" \
  | sudo tee /etc/apt/sources.list.d/mongodb-enterprise-7.0.list
sudo apt-get update
sudo apt-get install -y mongodb-enterprise
```

On RHEL/CentOS:

```bash
# Add Enterprise repo for RHEL 9
sudo tee /etc/yum.repos.d/mongodb-enterprise-7.0.repo << 'EOF'
[mongodb-enterprise-7.0]
name=MongoDB Enterprise Repository
baseurl=https://repo.mongodb.com/yum/redhat/9/mongodb-enterprise/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-7.0.asc
EOF
sudo dnf install -y mongodb-enterprise
```

## Perform the In-Place Switch

Stop the Community `mongod`, install Enterprise, and restart with the same data directory:

```bash
# Stop Community mongod
sudo systemctl stop mongod

# Install Enterprise (data directory unchanged)
sudo apt-get install -y mongodb-enterprise

# Restart with Enterprise
sudo systemctl start mongod

# Verify Enterprise is running
mongosh --eval "db.adminCommand({ buildInfo: 1 }).modules"
# Should output: [ 'enterprise' ]
```

## Configure Audit Logging

Enable audit logging in `mongod.conf` to capture all operations:

```yaml
# mongod.conf - enable audit log (Enterprise only)
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{ atype: { $in: ["authenticate", "createCollection", "dropCollection", "createIndex", "find", "insert", "update", "delete"] } }'
```

## Configure LDAP Authentication

```yaml
# mongod.conf - LDAP authentication (Enterprise only)
security:
  authorization: enabled
  ldap:
    servers: "ldap.example.com"
    transportSecurity: tls
    userToDNMapping:
      '[{ match: "(.+)", substitution: "cn={0},ou=users,dc=example,dc=com" }]'
    authz:
      queryTemplate: "ou=groups,dc=example,dc=com??sub?(member={USER})"

setParameter:
  authenticationMechanisms: "PLAIN"
```

## Enable Encrypted Storage at Rest

```yaml
# mongod.conf - WiredTiger encrypted storage (Enterprise only)
storage:
  dbPath: /var/lib/mongodb
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      encryptionKeyIdentifier: "localKey"

security:
  enableEncryption: true
  encryptionKeyFile: /etc/mongodb/keyfile
```

## Summary

Migrating from MongoDB Community to Enterprise is an in-place upgrade requiring only a package swap and configuration changes - no data migration needed. After the switch, enable audit logging for compliance, configure LDAP for centralized authentication, and consider encrypted storage at rest for regulated workloads. Validate Enterprise features are active by checking `db.adminCommand({ buildInfo: 1 }).modules`.
