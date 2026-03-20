# How to Configure MySQL SSL/TLS for Encrypted IPv4 Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SSL, TLS, IPv4, Encryption, Security, Database

Description: Enable SSL/TLS on MySQL to encrypt connections from remote IPv4 clients, generate certificates, require SSL for specific users, and verify encrypted connections.

## Introduction

MySQL connections over IPv4 can be intercepted on untrusted networks. SSL/TLS encrypts the data stream between client and server. MySQL 8.0 enables SSL by default if certificates exist; configuring it explicitly ensures it works correctly and can be required for specific users.

## Generating Certificates

```bash
# MySQL 8.0+ includes mysql_ssl_rsa_setup

sudo mysql_ssl_rsa_setup --uid=mysql

# Or generate manually with OpenSSL:

# CA key and certificate
openssl genrsa -out ca-key.pem 2048
openssl req -new -x509 -nodes -days 3650 \
  -key ca-key.pem -out ca.pem \
  -subj "/CN=MySQL CA"

# Server key and certificate
openssl genrsa -out server-key.pem 2048
openssl req -new -key server-key.pem -out server-req.pem \
  -subj "/CN=mysql-server"
openssl x509 -req -days 3650 -in server-req.pem \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -out server-cert.pem

# Client key and certificate
openssl genrsa -out client-key.pem 2048
openssl req -new -key client-key.pem -out client-req.pem \
  -subj "/CN=mysql-client"
openssl x509 -req -days 3650 -in client-req.pem \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -out client-cert.pem

# Copy to MySQL data directory
sudo cp ca.pem server-cert.pem server-key.pem /var/lib/mysql/
sudo chown mysql:mysql /var/lib/mysql/{ca,server-cert,server-key}.pem
sudo chmod 600 /var/lib/mysql/server-key.pem
```

## MySQL Server Configuration

```bash
# /etc/mysql/mysql.conf.d/mysqld.cnf

[mysqld]
bind-address = 203.0.113.10

# SSL certificates
ssl-ca     = /var/lib/mysql/ca.pem
ssl-cert   = /var/lib/mysql/server-cert.pem
ssl-key    = /var/lib/mysql/server-key.pem

# Require TLS 1.2 minimum
tls_version = TLSv1.2,TLSv1.3
```

```bash
sudo systemctl restart mysql

# Verify SSL is enabled
sudo mysql -e "SHOW VARIABLES LIKE 'have_ssl';"
# Expected: have_ssl  YES

sudo mysql -e "SHOW VARIABLES LIKE 'ssl_%';"
```

## Requiring SSL for Specific Users

```bash
sudo mysql -u root -p

-- Require SSL for a user from remote IP
CREATE USER 'secureapp'@'10.0.0.5' IDENTIFIED BY 'password' REQUIRE SSL;

-- Require SSL on existing user
ALTER USER 'appuser'@'10.0.0.5' REQUIRE SSL;

-- Require specific cipher (optional)
ALTER USER 'appuser'@'10.0.0.5' REQUIRE CIPHER 'ECDHE-RSA-AES128-GCM-SHA256';

-- Check SSL requirement
SELECT User, Host, ssl_type FROM mysql.user WHERE User='appuser';

FLUSH PRIVILEGES;
```

## Connecting with SSL from Client

```bash
# Connect requiring SSL
mysql -h 203.0.113.10 -u secureapp -p \
  --ssl-ca=/etc/mysql/ca.pem \
  --ssl-cert=/etc/mysql/client-cert.pem \
  --ssl-key=/etc/mysql/client-key.pem

# Verify SSL is in use
mysql> SHOW STATUS LIKE 'Ssl_cipher';
# Expected: Ssl_cipher  ECDHE-RSA-AES128-GCM-SHA256

# Verify from status
mysql> \s
# SSL: Cipher in use is ECDHE-RSA-AES128-GCM-SHA256
```

## Conclusion

Enable MySQL SSL/TLS by specifying `ssl-ca`, `ssl-cert`, and `ssl-key` in `mysqld.cnf`. Use `mysql_ssl_rsa_setup` to generate certificates automatically. Require SSL for sensitive users with `ALTER USER ... REQUIRE SSL`. Verify encryption is active with `SHOW STATUS LIKE 'Ssl_cipher'` after connecting. Always use TLS 1.2 or higher by setting `tls_version`.
