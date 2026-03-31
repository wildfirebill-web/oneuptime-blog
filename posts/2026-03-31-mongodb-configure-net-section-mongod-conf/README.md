# How to Configure the net Section in mongod.conf

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Configuration, Network, Security, mongod

Description: Learn how to configure the net section in mongod.conf to control port, bind IP, TLS mode, max connections, and compression settings for MongoDB.

---

The `net` section of `mongod.conf` controls how MongoDB listens for client connections. Misconfiguring this section is one of the most common causes of connectivity issues and security exposure. Understanding each option lets you lock down access while keeping the database reachable for legitimate clients.

## Basic Structure

The `net` section lives at the top level of the YAML configuration file.

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1
  maxIncomingConnections: 65536
  wireObjectCheck: true
  ipv6: false
```

## Changing the Listening Port

The default port is 27017. Change it to reduce noise from automated scanners.

```yaml
net:
  port: 27117
```

After changing the port, update all connection strings and firewall rules before restarting.

## Restricting Bind Addresses

`bindIp` controls which network interfaces MongoDB listens on. The default in modern versions is `127.0.0.1`, which means only local connections are accepted.

```yaml
net:
  bindIp: 127.0.0.1,10.0.1.50
```

To accept connections on all interfaces (not recommended in production), set it to `0.0.0.0`. Prefer listing specific IP addresses instead.

## Enabling TLS

The `tls` subsection under `net` enables encrypted connections.

```yaml
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/server.pem
    CAFile: /etc/ssl/mongodb/ca.pem
    allowConnectionsWithoutCertificates: false
```

Setting `mode: requireTLS` rejects all non-TLS connections. During a migration, use `mode: allowTLS` or `mode: preferTLS` to allow a mixed period.

## Setting Maximum Connections

Limit the number of simultaneous client connections to protect the server from connection exhaustion.

```yaml
net:
  maxIncomingConnections: 200
```

Set this value based on the number of application servers and their connection pool sizes. A typical formula is: `(app_servers * pool_size) + headroom`.

## Enabling Compression

Enable network-level compression to reduce bandwidth between the client and server.

```yaml
net:
  compression:
    compressors: snappy,zlib,zstd
```

The client and server negotiate the best mutually supported algorithm. `snappy` offers low CPU overhead; `zstd` offers higher compression ratios.

## Restarting and Verifying

After editing `mongod.conf`, restart the service and confirm the settings are active.

```bash
sudo systemctl restart mongod
mongosh --port 27117 --eval "db.adminCommand({ getCmdLineOpts: 1 })"
```

The `getCmdLineOpts` command returns the parsed configuration, which lets you verify that your changes took effect without inspecting the raw file.

## Summary

The `net` section in `mongod.conf` governs port binding, IP restrictions, TLS enforcement, connection limits, and compression. Restrict `bindIp` to specific addresses, enable `requireTLS` for encrypted traffic, and set `maxIncomingConnections` based on your application topology. Always restart `mongod` after changes and verify with `getCmdLineOpts`.
