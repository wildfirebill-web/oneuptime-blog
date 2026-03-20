# How to Configure ksqlDB to Listen on a Specific IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kafka, ksqlDB, IPv4, Streaming, Configuration, SQL

Description: Configure ksqlDB server to listen on a specific IPv4 address, connect to Kafka brokers over IPv4, and restrict access to the ksqlDB REST API.

## Introduction

ksqlDB is a streaming SQL engine built on Kafka Streams. It exposes a REST API for running queries and creating streams/tables. Configuring which IPv4 address ksqlDB listens on controls access to this API and enables network isolation.

## ksqlDB Server Configuration

```properties
# /etc/ksqldb/ksql-server.properties

# REST API listener — bind to specific IPv4
listeners=http://10.0.0.5:8088

# Or for HTTPS:
# listeners=https://10.0.0.5:8088

# Bootstrap server(s) — Kafka broker IPv4 addresses
bootstrap.servers=10.0.0.1:9092,10.0.0.2:9092,10.0.0.3:9092

# ksqlDB service URL (advertised to clients)
ksql.advertised.listener=http://10.0.0.5:8088

# KsqlDB state store and processing
ksql.streams.state.dir=/var/lib/ksqldb/data
```

## Security Configuration

```properties
# /etc/ksqldb/ksql-server.properties

# Require authentication for REST API
ksql.authentication.plugin.class=io.confluent.ksql.security.VertxBasicAuthCredential
ksql.authentication.basic.users.admin=AdminPass123
ksql.authentication.basic.users.readonly=ReadPass

# SSL for Kafka connections (if brokers use SSL)
security.protocol=SSL
ssl.truststore.location=/etc/ksqldb/ssl/kafka.client.truststore.jks
ssl.truststore.password=trustpass
```

## Running ksqlDB

```bash
# Start ksqlDB server
sudo systemctl start ksqldb-server

# Verify it's listening
sudo ss -tlnp | grep java | grep 8088
# Expected: 10.0.0.5:8088

# Check ksqlDB logs
sudo tail -20 /var/log/ksqldb/ksql.log
```

## Firewall Rules

```bash
# Allow ksqlDB REST API from app servers only
sudo ufw allow from 10.0.0.0/24 to any port 8088
sudo ufw deny 8088

# iptables
sudo iptables -A INPUT -p tcp --dport 8088 -s 10.0.0.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 8088 -j DROP
```

## Using the ksqlDB CLI

```bash
# Connect ksqlDB CLI to specific server IP
ksql http://10.0.0.5:8088

# Or with authentication:
ksql --user admin --password AdminPass123 http://10.0.0.5:8088

# In the CLI, run SQL:
ksql> CREATE STREAM orders (id BIGINT, product VARCHAR)
      WITH (KAFKA_TOPIC='orders', VALUE_FORMAT='JSON');

ksql> SELECT * FROM orders EMIT CHANGES;

# List streams
ksql> SHOW STREAMS;
```

## Using the REST API

```bash
# List all streams
curl -s http://10.0.0.5:8088/ksql \
  -H "Content-Type: application/vnd.ksql.v1+json" \
  -d '{"ksql": "SHOW STREAMS;"}' | python3 -m json.tool

# Run a push query
curl -s http://10.0.0.5:8088/query-stream \
  -H "Content-Type: application/vnd.ksql.v1+json" \
  -d '{"sql": "SELECT * FROM orders EMIT CHANGES;"}'

# Check server status
curl -s http://10.0.0.5:8088/info
```

## Conclusion

ksqlDB's `listeners` property (not `listener.name`) defines the REST API binding address. Use `http://ip:8088` format to bind to a specific IPv4. Set `bootstrap.servers` to list all Kafka broker IPv4 addresses. Restrict the REST API port with firewall rules, and enable basic authentication for production deployments. The ksqlDB CLI and REST API use the same port.
