# How to Use mysql_ssl_rsa_setup for SSL Configuration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SSL, Security, TLS, Certificate

Description: Learn how to use mysql_ssl_rsa_setup to generate SSL certificates and RSA key pairs for securing MySQL client-server connections.

---

## What is mysql_ssl_rsa_setup?

`mysql_ssl_rsa_setup` is a utility that generates the SSL certificate and RSA key files needed to enable encrypted connections in MySQL. It creates a self-signed Certificate Authority (CA), server certificates, and client certificates, as well as RSA key pairs used for password exchange over unencrypted connections. This tool simplifies the process of enabling SSL without requiring you to run OpenSSL commands manually.

Note: In MySQL 8.0.34+, this utility is deprecated in favor of automatic SSL setup. However, it remains useful on older 8.0 releases and MySQL 5.7.

## Prerequisites

Ensure OpenSSL is installed on the system:

```bash
openssl version
```

Also verify that the `mysql_ssl_rsa_setup` binary is available:

```bash
which mysql_ssl_rsa_setup
```

## Generating SSL Files

Run the utility as root or with sudo, pointing to the MySQL data directory:

```bash
mysql_ssl_rsa_setup --datadir=/var/lib/mysql
```

This creates the following files in `/var/lib/mysql`:

```text
ca.pem          - Certificate Authority certificate
ca-key.pem      - CA private key (keep secret)
server-cert.pem - Server certificate
server-key.pem  - Server private key (keep secret)
client-cert.pem - Client certificate
client-key.pem  - Client private key
public_key.pem  - RSA public key for password exchange
private_key.pem - RSA private key for password exchange
```

## Generating Files to a Custom Directory

If you want to store certificates outside the data directory:

```bash
mkdir -p /etc/mysql/ssl
mysql_ssl_rsa_setup --datadir=/etc/mysql/ssl --uid=mysql
```

The `--uid` option sets file ownership to the `mysql` user so the server can read them.

## Configuring MySQL to Use the SSL Files

Add the SSL paths to `/etc/mysql/my.cnf` under `[mysqld]`:

```text
[mysqld]
ssl-ca=/var/lib/mysql/ca.pem
ssl-cert=/var/lib/mysql/server-cert.pem
ssl-key=/var/lib/mysql/server-key.pem
```

Restart MySQL:

```bash
systemctl restart mysql
```

## Verifying SSL is Active

Connect and check the SSL status:

```sql
SHOW VARIABLES LIKE '%ssl%';
```

Expected output includes:

```text
have_ssl    | YES
ssl_ca      | /var/lib/mysql/ca.pem
ssl_cert    | /var/lib/mysql/server-cert.pem
```

## Connecting a Client with SSL

```bash
mysql -u myuser -p \
  --ssl-ca=/var/lib/mysql/ca.pem \
  --ssl-cert=/var/lib/mysql/client-cert.pem \
  --ssl-key=/var/lib/mysql/client-key.pem \
  -h 127.0.0.1
```

## Enforcing SSL for a User Account

```sql
ALTER USER 'appuser'@'%' REQUIRE SSL;
```

To require a specific cipher:

```sql
ALTER USER 'appuser'@'%' REQUIRE CIPHER 'ECDHE-RSA-AES256-GCM-SHA384';
```

## Verifying the Current Connection is Encrypted

After connecting, run:

```sql
STATUS;
```

Look for `SSL: Cipher in use` in the output to confirm the connection is encrypted.

## Summary

`mysql_ssl_rsa_setup` provides a quick way to bootstrap SSL/TLS for MySQL by generating all required certificate and key files in one command. After generating the files, update your MySQL configuration to reference them, restart the server, and optionally enforce SSL at the user level. For production environments, replace self-signed certificates with ones issued by a trusted CA.
