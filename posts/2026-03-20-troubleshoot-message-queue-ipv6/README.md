# How to Troubleshoot Message Queue IPv6 Connectivity Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, RabbitMQ, Kafka, NATS, Message Queue, Troubleshooting

Description: A systematic guide to diagnosing and resolving IPv6 connectivity problems with message queue systems including RabbitMQ, Kafka, NATS, and other messaging infrastructure.

## Common IPv6 Message Queue Errors

```text
# RabbitMQ AMQP client

Error: connect ECONNREFUSED [2001:db8::10]:5672

# Kafka producer
[2001:db8::10]:9092 -- BROKER_NOT_AVAILABLE

# NATS client
nats: no servers available for connection

# ZooKeeper (used by Kafka)
KeeperException: ConnectionLoss for /brokers/ids
```

## Step 1: Verify Services Are Listening on IPv6

```bash
# Check all message queue services
echo "=== Checking Message Queue IPv6 Listeners ==="
ss -6 -tlnp | grep -E "(5672|15672|9092|4222|2181|11211)"

# RabbitMQ AMQP (5672) and Management UI (15672)
ss -6 -tlnp | grep ':5672'
ss -6 -tlnp | grep ':15672'

# Kafka (9092)
ss -6 -tlnp | grep ':9092'

# NATS (4222)
ss -6 -tlnp | grep ':4222'

# ZooKeeper (2181)
ss -6 -tlnp | grep ':2181'
```

## Step 2: Test Network Connectivity

```bash
# ICMP reachability
ping6 -c 4 2001:db8::10

# TCP port tests for common MQ services
nc -6 -zv 2001:db8::10 5672   # RabbitMQ
nc -6 -zv 2001:db8::10 9092   # Kafka
nc -6 -zv 2001:db8::10 4222   # NATS
nc -6 -zv 2001:db8::10 2181   # ZooKeeper

# Check route to IPv6 host
traceroute6 2001:db8::10

# Verify local IPv6 addresses
ip -6 addr show
```

## Step 3: Check Firewall Rules

```bash
# List IPv6 firewall rules
ip6tables -L -n -v | grep -E "(5672|9092|4222|2181)"

# Quick check for specific ports
ip6tables -L INPUT -n | grep "5672"

# firewalld
firewall-cmd --list-ports
firewall-cmd --list-rich-rules

# Temporarily allow ports for testing
ip6tables -I INPUT -p tcp --dport 5672 -j ACCEPT
ip6tables -I INPUT -p tcp --dport 9092 -j ACCEPT
ip6tables -I INPUT -p tcp --dport 4222 -j ACCEPT
```

## Step 4: Check Configuration Files

```bash
# RabbitMQ - check listener configuration
grep -i "listeners\|tcp\|ip" /etc/rabbitmq/rabbitmq.conf 2>/dev/null
grep -i "tcp_listeners\|listen" /etc/rabbitmq/advanced.config 2>/dev/null

# Kafka - check listeners and advertised.listeners
grep -i "listeners\|advertised" /etc/kafka/server.properties

# NATS - check host/listen directives
grep -i "host\|listen\|port" /etc/nats/nats-server.conf

# ZooKeeper - check clientPortAddress
grep -i "clientPortAddress\|server\." /etc/zookeeper/conf/zoo.cfg
```

## Step 5: Check JVM IPv6 Configuration (Java-based MQ)

```bash
# Check if Kafka JVM has IPv6 flag
ps aux | grep -E "(kafka|zookeeper|rabbitmq)" | grep "preferIPv6"

# Check JVM options files
cat /etc/kafka/kafka-env.sh | grep -i ipv6
cat /etc/kafka/jvm.options | grep -i ipv6

# Fix: Add JVM flag
# For Kafka: Add to /etc/kafka/kafka-env.sh
echo 'export KAFKA_OPTS="-Djava.net.preferIPv6Addresses=true $KAFKA_OPTS"' >> /etc/kafka/kafka-env.sh

# For ZooKeeper
echo 'export SERVER_JVMFLAGS="-Djava.net.preferIPv6Addresses=true $SERVER_JVMFLAGS"' >> /etc/zookeeper/conf/java.env
```

## Step 6: Check Advertised vs. Actual Addresses

```bash
# Kafka: advertised.listeners must match what clients can reach
# Check what brokers are advertising
kafka-broker-api-versions.sh --bootstrap-server [2001:db8::10]:9092 2>&1 | head -20

# RabbitMQ: Check node address in cluster
rabbitmqctl cluster_status

# NATS: Check routing info
curl -6 http://[2001:db8::10]:8222/routez
curl -6 http://[2001:db8::10]:8222/varz | grep listen_addresses
```

## Common Fixes Reference

```bash
# Fix 1: Service not listening on IPv6 - update config and restart
# RabbitMQ: Set listeners.tcp.1 = 2001:db8::10:5672
# Kafka: Set listeners=PLAINTEXT://[2001:db8::10]:9092
# NATS: Set host: "2001:db8::10"

# Fix 2: Firewall blocking
ip6tables -A INPUT -p tcp --dport 5672 -j ACCEPT  # RabbitMQ
ip6tables -A INPUT -p tcp --dport 9092 -j ACCEPT  # Kafka

# Fix 3: JVM not preferring IPv6
export JAVA_OPTS="-Djava.net.preferIPv6Addresses=true"

# Fix 4: Kafka advertised listeners using hostname that resolves to IPv4
# Change: advertised.listeners=PLAINTEXT://hostname:9092
# To:     advertised.listeners=PLAINTEXT://[2001:db8::10]:9092

# Fix 5: ZooKeeper not accessible over IPv6
# Set: clientPortAddress=2001:db8::10 in zoo.cfg
```

## Summary

Troubleshoot message queue IPv6 issues systematically: (1) verify the service binds to IPv6 with `ss -6 -tlnp`; (2) test port reachability with `nc -6 -zv`; (3) check firewall rules with `ip6tables -L`; (4) inspect configuration files for IPv6 bind addresses; (5) for Java-based queues (Kafka, ZooKeeper), add `-Djava.net.preferIPv6Addresses=true` to JVM options; (6) ensure advertised addresses match what clients can actually reach.
