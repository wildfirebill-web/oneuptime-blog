# How to Configure MongoDB Connection Strings (URI Format)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Connection, URI, Configuration, Driver

Description: Learn the MongoDB URI connection string format with all key options including authentication, TLS, replica sets, and connection pool parameters.

---

The MongoDB connection string URI is the standard way to configure how applications connect to MongoDB. A correctly structured URI handles authentication, TLS, timeouts, and connection pool settings in a single string.

## Basic URI Structure

```text
mongodb://[username:password@]host[:port][/database][?options]
```

Examples:

```text
mongodb://localhost:27017
mongodb://localhost:27017/myDatabase
mongodb://user:pass@localhost:27017/myDatabase?authSource=admin
```

## Standard vs SRV Format

The standard format specifies hosts directly:

```text
mongodb://host1:27017,host2:27017,host3:27017/myDB?replicaSet=rs0
```

The SRV format (for MongoDB Atlas and DNS-managed clusters) looks up hosts via DNS:

```text
mongodb+srv://user:pass@cluster0.abcde.mongodb.net/myDB
```

## Authentication Options

Specify credentials and authentication mechanism:

```text
mongodb://appUser:s3cureP%40ss@localhost:27017/myDB?authSource=admin&authMechanism=SCRAM-SHA-256
```

Note: special characters in passwords must be percent-encoded (e.g., `@` becomes `%40`, `:` becomes `%3A`).

Common `authMechanism` values:

```text
SCRAM-SHA-1       (MongoDB 3.x default)
SCRAM-SHA-256     (MongoDB 4.0+ default)
MONGODB-X509      (certificate-based)
GSSAPI            (Kerberos)
PLAIN             (LDAP)
```

## TLS/SSL Options

```text
mongodb://user:pass@host:27017/myDB?tls=true&tlsCAFile=/path/to/ca.pem&tlsCertificateKeyFile=/path/to/client.pem
```

Key TLS options:

```text
tls=true                          enable TLS
tlsCAFile=<path>                  CA certificate
tlsCertificateKeyFile=<path>      client cert+key PEM
tlsAllowInvalidCertificates=true  disable cert validation (dev only)
tlsAllowInvalidHostnames=true     disable hostname check (dev only)
```

## Replica Set Connection

```text
mongodb://host1:27017,host2:27017,host3:27017/myDB?replicaSet=rs0&readPreference=secondaryPreferred
```

`readPreference` options:

```text
primary               (default) read from primary only
primaryPreferred      prefer primary, fall back to secondary
secondary             read from secondary only
secondaryPreferred    prefer secondary, fall back to primary
nearest               lowest network latency
```

## Connection Pool and Timeout Options

```text
mongodb://host:27017/myDB?maxPoolSize=50&minPoolSize=5&maxIdleTimeMS=30000&connectTimeoutMS=10000&socketTimeoutMS=30000&serverSelectionTimeoutMS=15000
```

Key parameters:

```text
maxPoolSize           maximum connections in pool (default: 100)
minPoolSize           minimum idle connections maintained
maxIdleTimeMS         max time a connection can be idle before closing
connectTimeoutMS      timeout for establishing a new connection
socketTimeoutMS       timeout for socket read/write operations
serverSelectionTimeoutMS  how long the driver waits to find a server
```

## Write Concern in the URI

```text
mongodb://host:27017/myDB?w=majority&j=true&wtimeoutMS=5000
```

## Compressing Wire Traffic

```text
mongodb://host:27017/myDB?compressors=snappy,zstd
```

## Full Production Example

```text
mongodb://appUser:S3cure%21Pass@mongo1:27017,mongo2:27017,mongo3:27017/prodDB?replicaSet=rs0&authSource=admin&tls=true&tlsCAFile=/etc/mongo/ca.pem&maxPoolSize=50&connectTimeoutMS=10000&serverSelectionTimeoutMS=15000&w=majority&readPreference=secondaryPreferred
```

## Testing a Connection String

```bash
mongosh "mongodb://user:pass@host:27017/myDB?authSource=admin" \
  --eval "db.runCommand({ ping: 1 })"
```

## Summary

The MongoDB URI format packs authentication, TLS, replica set configuration, and connection pool tuning into a single string. Use `mongodb+srv://` for Atlas clusters, percent-encode special characters in credentials, and always set `serverSelectionTimeoutMS` in production to control how long the driver waits when no server is available.
