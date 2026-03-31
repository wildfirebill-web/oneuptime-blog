# How to Enable TLS for ClickHouse Client Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, TLS, Security, Encryption, HTTPS, Certificate

Description: Learn how to enable TLS encryption for ClickHouse client connections on the HTTP and native TCP interfaces to protect data in transit.

---

By default, ClickHouse communicates over plaintext connections. Enabling TLS ensures that all data, including credentials and query results, is encrypted in transit between clients and the ClickHouse server.

## Generating or Obtaining Certificates

For production, use certificates from a trusted CA. For testing, generate self-signed certificates:

```bash
# Generate CA key and certificate
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=ClickHouse CA"

# Generate server key and CSR
openssl genrsa -out server.key 4096
openssl req -new -key server.key -out server.csr \
  -subj "/CN=clickhouse.example.com"

# Sign with CA
openssl x509 -req -days 365 -in server.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt
```

Copy certificates to ClickHouse:

```bash
cp server.crt /etc/clickhouse-server/server.crt
cp server.key /etc/clickhouse-server/server.key
cp ca.crt /etc/clickhouse-server/ca.crt
chown clickhouse:clickhouse /etc/clickhouse-server/*.crt /etc/clickhouse-server/*.key
chmod 600 /etc/clickhouse-server/server.key
```

## Configuring TLS in config.xml

Add TLS configuration to `config.xml`:

```text
<!-- HTTPS interface (port 8443) -->
<https_port>8443</https_port>

<!-- Native TLS interface (port 9440) -->
<tcp_port_secure>9440</tcp_port_secure>

<openSSL>
  <server>
    <certificateFile>/etc/clickhouse-server/server.crt</certificateFile>
    <privateKeyFile>/etc/clickhouse-server/server.key</privateKeyFile>
    <caConfig>/etc/clickhouse-server/ca.crt</caConfig>
    <verificationMode>none</verificationMode>
    <loadDefaultCAFile>true</loadDefaultCAFile>
    <cacheSessions>true</cacheSessions>
    <disableProtocols>sslv2,sslv3</disableProtocols>
    <preferServerCiphers>true</preferServerCiphers>
  </server>
</openSSL>
```

Restart ClickHouse after changes:

```bash
systemctl restart clickhouse-server
```

## Connecting via HTTPS

Test the HTTPS connection with curl:

```bash
curl --cacert /etc/clickhouse-server/ca.crt \
  -u default:password \
  'https://localhost:8443/?query=SELECT+1'
```

## Connecting via Native TLS

Using the `clickhouse-client`:

```bash
clickhouse-client \
  --secure \
  --port 9440 \
  --host localhost \
  --user default \
  --password password
```

## Disabling Plaintext Ports (Optional)

For maximum security, disable unencrypted ports by commenting them out in `config.xml`:

```text
<!-- <http_port>8123</http_port> -->
<!-- <tcp_port>9000</tcp_port> -->
```

Keep only the secure ports active.

## Configuring Clients to Trust the CA

For Python clients using `clickhouse-driver`:

```python
from clickhouse_driver import Client

client = Client(
    host='clickhouse.example.com',
    port=9440,
    secure=True,
    ca_certs='/etc/ssl/certs/ca.crt',
    user='default',
    password='password'
)
result = client.execute('SELECT 1')
```

For JDBC connections:

```text
jdbc:clickhouse://clickhouse.example.com:8443/default?ssl=true&sslmode=strict&sslrootcert=/path/to/ca.crt
```

## Enforcing Strong Cipher Suites

Restrict to modern, strong cipher suites:

```text
<openSSL>
  <server>
    ...
    <cipherList>ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384</cipherList>
    <disableProtocols>sslv2,sslv3,tlsv1,tlsv1_1</disableProtocols>
  </server>
</openSSL>
```

## Summary

Enabling TLS for ClickHouse client connections requires generating certificates, configuring the `openSSL` section in `config.xml`, and enabling the HTTPS and native TLS ports. Update all clients to use secure connections and consider disabling plaintext ports in production environments.
