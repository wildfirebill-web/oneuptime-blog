# How to Configure RabbitMQ TLS/SSL Listeners on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RabbitMQ, TLS, SSL, IPv4, Encryption, AMQPS, Messaging

Description: Enable TLS/SSL on RabbitMQ AMQP listeners for encrypted IPv4 connections, configure certificates, and connect clients using AMQPS on port 5671.

## Introduction

RabbitMQ AMQP without TLS sends messages in plaintext. AMQPS (AMQP with TLS) on port 5671 encrypts all traffic. This is essential for production deployments where RabbitMQ is accessed over untrusted networks.

## Generating Certificates

```bash
# CA certificate

openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt \
  -subj "/CN=RabbitMQ CA"

# Server certificate
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr \
  -subj "/CN=10.0.0.5"
openssl x509 -req -days 3650 -in server.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt

# Copy to RabbitMQ config directory
sudo cp ca.crt server.crt server.key /etc/rabbitmq/
sudo chown rabbitmq:rabbitmq /etc/rabbitmq/{ca.crt,server.crt,server.key}
sudo chmod 600 /etc/rabbitmq/server.key
```

## TLS Configuration

```bash
# /etc/rabbitmq/rabbitmq.conf

# Plain AMQP (can keep for internal connections)
listeners.tcp.1 = 127.0.0.1:5672

# AMQPS listener on specific IPv4
listeners.ssl.1 = 10.0.0.5:5671

# TLS/SSL settings
ssl_options.cacertfile = /etc/rabbitmq/ca.crt
ssl_options.certfile   = /etc/rabbitmq/server.crt
ssl_options.keyfile    = /etc/rabbitmq/server.key
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = false    # Set to true for mutual TLS

# TLS version restrictions
ssl_options.versions.1 = tlsv1.2
ssl_options.versions.2 = tlsv1.3

# Strong ciphers
ssl_options.ciphers.1 = ECDHE-RSA-AES128-GCM-SHA256
ssl_options.ciphers.2 = ECDHE-RSA-AES256-GCM-SHA384
```

```bash
sudo systemctl restart rabbitmq-server

# Verify TLS listener
sudo rabbitmq-diagnostics listeners
# Should show: ssl port 5671 on 10.0.0.5
```

## Firewall for AMQPS

```bash
# Allow AMQPS from app servers
sudo ufw allow from 10.0.0.0/24 to any port 5671
sudo ufw deny 5671

# If disabling plain AMQP from remote:
# sudo ufw deny 5672 from 0.0.0.0/0
```

## Connecting Clients with TLS

```bash
# Python (pika):
import pika
import ssl

ssl_context = ssl.create_default_context(cafile="/etc/rabbitmq/ca.crt")
ssl_options = pika.SSLOptions(ssl_context, "10.0.0.5")
params = pika.ConnectionParameters(
    host="10.0.0.5",
    port=5671,
    ssl_options=ssl_options,
    credentials=pika.PlainCredentials("appuser", "apppass")
)
conn = pika.BlockingConnection(params)

# Node.js (amqplib):
const amqp = require('amqplib');
amqp.connect({
  hostname: '10.0.0.5',
  port: 5671,
  username: 'appuser',
  password: 'apppass',
  protocol: 'amqps',
  ca: [require('fs').readFileSync('/etc/rabbitmq/ca.crt')]
});
```

## Verifying TLS

```bash
# Test TLS handshake
openssl s_client -connect 10.0.0.5:5671 -CAfile /etc/rabbitmq/ca.crt

# Check TLS status in RabbitMQ
sudo rabbitmq-diagnostics tls_status

# View TLS connections
sudo rabbitmqctl list_connections ssl peer_host user
```

## Conclusion

Enable RabbitMQ AMQPS by adding `listeners.ssl.1 = ip:5671` and configuring `ssl_options.*` in `rabbitmq.conf`. Restrict TLS versions to 1.2 and 1.3, and use strong cipher suites. Clients connect on port 5671 using `amqps://` with the CA certificate for server verification. For internal traffic, keep plain AMQP on localhost and require AMQPS for all remote connections.
