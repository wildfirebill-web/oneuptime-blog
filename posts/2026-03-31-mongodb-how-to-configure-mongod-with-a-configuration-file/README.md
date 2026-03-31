# How to Configure mongod with a Configuration File

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mongodb, Configuration, Devops, Database Administration

Description: Learn how to configure the MongoDB mongod process using a YAML configuration file to control storage, networking, logging, and security settings.

---

## Overview

MongoDB's `mongod` process can be configured through command-line flags or through a dedicated configuration file. Using a configuration file is recommended for production deployments because it centralizes all settings, makes changes auditable, and simplifies process management.

By default, MongoDB looks for its configuration file at `/etc/mongod.conf` on Linux systems. You can specify a custom path using the `--config` flag.

## The Configuration File Format

MongoDB configuration files use YAML syntax. Settings are organized into sections that correspond to different aspects of the server:

```yaml
# /etc/mongod.conf

storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 127.0.0.1

processManagement:
  timeZoneInfo: /usr/share/zoneinfo
```

## Storage Configuration

The `storage` section controls where MongoDB stores data and how it manages disk I/O:

```yaml
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true
```

Key options:
- `dbPath` - directory where MongoDB stores its data files
- `journal.enabled` - enables write-ahead journaling for durability
- `engine` - storage engine (wiredTiger is the default since MongoDB 3.2)
- `wiredTiger.engineConfig.cacheSizeGB` - amount of RAM for the WiredTiger cache

## Network Configuration

The `net` section controls how MongoDB listens for connections:

```yaml
net:
  port: 27017
  bindIp: 0.0.0.0
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb.pem
    CAFile: /etc/ssl/ca.pem
```

- `bindIp` - comma-separated list of IP addresses to bind to
- `port` - TCP port for client connections
- `tls.mode` - TLS mode: disabled, allowTLS, preferTLS, or requireTLS

## Logging Configuration

```yaml
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  verbosity: 0
  component:
    replication:
      verbosity: 1
    query:
      verbosity: 0
```

## Security Configuration

```yaml
security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile
  javascriptEnabled: false
```

- `authorization` - enable role-based access control
- `keyFile` - path to keyfile used for replica set authentication
- `javascriptEnabled` - disable server-side JavaScript for security hardening

## Replica Set Configuration

```yaml
replication:
  replSetName: "rs0"
  oplogSizeMB: 1024
```

## Starting mongod with a Config File

```bash
# Use a specific config file
mongod --config /etc/mongod.conf

# Short flag form
mongod -f /etc/mongod.conf
```

## Converting Command-Line Options to Config File

If you are running mongod with command-line flags, convert them to config file format. For example:

```bash
# Command-line version
mongod --dbpath /var/lib/mongodb --port 27017 --logpath /var/log/mongodb/mongod.log --auth
```

Becomes this config file:

```yaml
storage:
  dbPath: /var/lib/mongodb
net:
  port: 27017
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
security:
  authorization: enabled
```

## A Full Production Configuration Example

```yaml
# /etc/mongod.conf - Production configuration

storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 127.0.0.1,10.0.0.5
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb.pem

security:
  authorization: enabled
  javascriptEnabled: false

replication:
  replSetName: "rs-prod"
  oplogSizeMB: 2048

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

operationProfiling:
  slowOpThresholdMs: 100
  mode: slowOp
```

## Summary

Using a configuration file for `mongod` centralizes all settings in a single readable YAML document. The file is divided into sections covering storage, networking, logging, security, and replication. This approach makes deployments reproducible and easier to manage compared to long command-line argument strings.
