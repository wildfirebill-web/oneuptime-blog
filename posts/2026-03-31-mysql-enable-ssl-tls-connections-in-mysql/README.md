# How to Enable SSL/TLS Connections in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, SSL, TLS, Encryption, Database Security

Description: Learn how to enable and configure SSL/TLS encrypted connections in MySQL to protect data in transit between clients and the server.

---

MySQL supports SSL/TLS to encrypt all data transmitted between the client and server. Without encryption, credentials and query results travel as plaintext over the network. This guide walks through enabling SSL/TLS from scratch.

## Check Current SSL Status

Before making changes, verify whether SSL is already active:

```sql
SHOW VARIABLES LIKE '%ssl%';
SHOW VARIABLES LIKE 'have_ssl';
```

If `have_ssl` shows `YES` and `ssl_cert` has a value, SSL is already configured. If it shows `DISABLED`, you need to enable it.

## Generate SSL Certificates

MySQL 8.0+ ships with `mysql_ssl_rsa_setup` to auto-generate all required certificates:

```bash
sudo mysql_ssl_rsa_setup --datadir=/var/lib/mysql
```

This creates the following files in the data directory:

```text
ca.pem          - Certificate Authority certificate
ca-key.pem      - CA private key
server-cert.pem - Server certificate
server-key.pem  - Server private key
client-cert.pem - Client certificate
client-key.pem  - Client private key
```

## Configure MySQL to Use SSL

Edit the MySQL configuration file (typically `/etc/mysql/mysql.conf.d/mysqld.cnf` or `/etc/my.cnf`) to point to the generated certificates:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Add or update the `[mysqld]` section:

```text
[mysqld]
ssl-ca=/var/lib/mysql/ca.pem
ssl-cert=/var/lib/mysql/server-cert.pem
ssl-key=/var/lib/mysql/server-key.pem
```

Restart MySQL to apply the changes:

```bash
sudo systemctl restart mysql
```

## Verify SSL is Active on the Server

After restarting, confirm SSL is enabled:

```sql
SHOW VARIABLES LIKE 'have_ssl';
SHOW VARIABLES LIKE 'ssl_cert';
```

Expected output:

```text
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| have_ssl      | YES   |
+---------------+-------+
```

## Connect as a Client Using SSL

Pass the certificate paths when connecting from the MySQL CLI:

```bash
mysql -u myuser -p \
  --ssl-ca=/var/lib/mysql/ca.pem \
  --ssl-cert=/var/lib/mysql/client-cert.pem \
  --ssl-key=/var/lib/mysql/client-key.pem \
  -h 192.168.1.10
```

To verify the current session is using SSL:

```sql
SHOW STATUS LIKE 'Ssl_cipher';
```

A non-empty cipher name confirms the connection is encrypted.

## Require SSL for Specific Users

You can enforce SSL for individual user accounts so they cannot connect without it:

```sql
ALTER USER 'appuser'@'%' REQUIRE SSL;
```

Or require a valid client certificate:

```sql
ALTER USER 'appuser'@'%' REQUIRE X509;
```

Check the requirement:

```sql
SELECT user, host, ssl_type FROM mysql.user WHERE user = 'appuser';
```

## Enforce SSL Server-Wide

To reject all plaintext connections at the server level, add this to `[mysqld]`:

```text
[mysqld]
require_secure_transport = ON
```

After restarting, any client that does not negotiate TLS will receive an error:

```text
ERROR 3159 (HY000): Connections using insecure transport are prohibited
while --require_secure_transport=ON.
```

## Verify TLS Version in Use

MySQL supports TLS 1.2 and TLS 1.3. To check which version the current session is using:

```sql
SHOW STATUS LIKE 'Ssl_version';
```

To restrict to TLS 1.3 only, add to `[mysqld]`:

```text
[mysqld]
tls_version = TLSv1.3
```

## Summary

Enabling SSL/TLS in MySQL involves generating certificates with `mysql_ssl_rsa_setup`, referencing them in `mysqld.cnf`, and restarting the server. Clients connect using the `--ssl-ca`, `--ssl-cert`, and `--ssl-key` flags. For tighter security, use `REQUIRE SSL` on individual accounts or set `require_secure_transport = ON` to enforce encryption for all connections.
