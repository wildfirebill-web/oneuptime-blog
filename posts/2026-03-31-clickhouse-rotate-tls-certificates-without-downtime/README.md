# How to Rotate TLS Certificates in ClickHouse Without Downtime

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, TLS, Certificate Rotation, Security, Zero Downtime

Description: Step-by-step guide to rotating TLS certificates in ClickHouse without restarting the server or dropping active connections.

---

Rotating TLS certificates without downtime is critical for production ClickHouse clusters. ClickHouse supports hot-reload of TLS configuration, allowing you to update certificates without stopping the server.

## Generate New Certificates

Use `openssl` to generate a new private key and CSR, then sign with your CA:

```bash
openssl genrsa -out /etc/clickhouse-server/server-new.key 4096
openssl req -new -key /etc/clickhouse-server/server-new.key \
  -out /etc/clickhouse-server/server-new.csr \
  -subj "/CN=clickhouse.example.com"
openssl x509 -req -in /etc/clickhouse-server/server-new.csr \
  -CA /etc/clickhouse-server/ca.crt -CAkey /etc/clickhouse-server/ca.key \
  -CAcreateserial -out /etc/clickhouse-server/server-new.crt -days 365
```

## Backup Existing Certificates

```bash
cp /etc/clickhouse-server/server.crt /etc/clickhouse-server/server.crt.bak
cp /etc/clickhouse-server/server.key /etc/clickhouse-server/server.key.bak
```

## Update the Config to Reference New Files

Edit `/etc/clickhouse-server/config.xml`:

```xml
<openSSL>
  <server>
    <certificateFile>/etc/clickhouse-server/server-new.crt</certificateFile>
    <privateKeyFile>/etc/clickhouse-server/server-new.key</privateKeyFile>
    <caConfig>/etc/clickhouse-server/ca.crt</caConfig>
    <verificationMode>relaxed</verificationMode>
  </server>
</openSSL>
```

## Reload Without Restart

ClickHouse supports SIGHUP-based config reload. This reloads the TLS configuration without dropping connections:

```bash
sudo kill -HUP $(pidof clickhouse-server)
```

Or use the system command:

```bash
echo "SYSTEM RELOAD CONFIG" | clickhouse-client -u admin --password secret
```

## Verify New Certificate is Active

Use `openssl s_client` to confirm the new certificate is served:

```bash
echo | openssl s_client -connect clickhouse.example.com:9440 2>/dev/null \
  | openssl x509 -noout -dates
```

## Rotate Client Certificates for Internode Communication

In a replicated cluster, also update the `<client>` section of `openSSL` in `config.xml` on each replica:

```xml
<openSSL>
  <client>
    <certificateFile>/etc/clickhouse-server/client-new.crt</certificateFile>
    <privateKeyFile>/etc/clickhouse-server/client-new.key</privateKeyFile>
    <caConfig>/etc/clickhouse-server/ca.crt</caConfig>
  </client>
</openSSL>
```

Reload each node one at a time, verifying replication health between rotations:

```sql
SELECT host_name, is_active FROM system.clusters;
```

## Summary

ClickHouse TLS certificates can be rotated without downtime by updating certificate file paths in `config.xml` and sending a SIGHUP or running `SYSTEM RELOAD CONFIG`. For clustered deployments, rotate one node at a time and verify replication health before proceeding to the next node.
