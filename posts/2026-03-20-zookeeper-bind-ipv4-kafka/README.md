# How to Set Up ZooKeeper to Bind to Specific IPv4 Addresses for Kafka

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ZooKeeper, Kafka, IPv4, Configuration, Distributed Systems, Networking

Description: Configure Apache ZooKeeper to bind to specific IPv4 addresses for Kafka coordination, set quorum peer addresses, configure firewall rules, and secure the ZooKeeper ensemble.

## Introduction

ZooKeeper is used by older Kafka deployments (pre-KRaft) for cluster coordination. By default ZooKeeper binds to all interfaces. In production, bind it to specific IPv4 addresses to limit exposure and ensure Kafka brokers connect on the correct network interface.

## ZooKeeper Configuration

```properties
# /opt/zookeeper/conf/zoo.cfg
# or /etc/zookeeper/conf/zoo.cfg

# Data directory
dataDir=/var/lib/zookeeper

# Client port binding — specific IPv4 address
# clientPortAddress binds the client port to a specific IP
clientPortAddress=10.0.0.10
clientPort=2181

# Admin server (ZooKeeper 3.5+)
admin.serverAddress=127.0.0.1
admin.serverPort=8080

# Session timeout bounds (milliseconds)
minSessionTimeout=4000
maxSessionTimeout=40000

# Tick time (fundamental time unit in ms)
tickTime=2000
initLimit=10
syncLimit=5

# Snapshot retention
autopurge.snapRetainCount=3
autopurge.purgeInterval=1
```

## ZooKeeper Ensemble (3-node cluster)

```properties
# /opt/zookeeper/conf/zoo.cfg (same on all 3 nodes)

clientPortAddress=10.0.0.10   # Each node uses its own IP here
clientPort=2181

# Quorum peers: server.ID=hostname:peerPort:leaderPort
# peerPort: inter-peer communication
# leaderPort: leader election
server.1=10.0.0.10:2888:3888
server.2=10.0.0.11:2888:3888
server.3=10.0.0.12:2888:3888

dataDir=/var/lib/zookeeper
tickTime=2000
initLimit=10
syncLimit=5
```

```bash
# Set myid file on each node (must match server.N in config)
# Node 1:
echo "1" > /var/lib/zookeeper/myid

# Node 2:
echo "2" > /var/lib/zookeeper/myid

# Node 3:
echo "3" > /var/lib/zookeeper/myid
```

## Binding Quorum Ports to Specific IPs

For ZooKeeper 3.5.7+, you can specify the IP for quorum communication:

```properties
# zoo.cfg
# Bind quorum ports to specific interface
server.1=10.0.0.10:2888:3888;10.0.0.10:2181
server.2=10.0.0.11:2888:3888;10.0.0.11:2181
server.3=10.0.0.12:2888:3888;10.0.0.12:2181

# The format is:
# server.ID=quorumIP:peerPort:leaderPort;clientIP:clientPort
```

## Kafka Connection to ZooKeeper

```properties
# /opt/kafka/config/server.properties

# Connect Kafka to ZooKeeper ensemble by IP
zookeeper.connect=10.0.0.10:2181,10.0.0.11:2181,10.0.0.12:2181/kafka

# ZooKeeper session timeout
zookeeper.session.timeout.ms=18000

# ZooKeeper connection timeout
zookeeper.connection.timeout.ms=18000
```

## Firewall Rules

```bash
# Allow Kafka brokers to reach ZooKeeper client port
KAFKA_BROKERS="10.0.0.20 10.0.0.21 10.0.0.22"
for broker in $KAFKA_BROKERS; do
  sudo iptables -A INPUT -s $broker -p tcp --dport 2181 -j ACCEPT
done

# Allow ZooKeeper quorum communication between peers
ZK_NODES="10.0.0.10 10.0.0.11 10.0.0.12"
for node in $ZK_NODES; do
  sudo iptables -A INPUT -s $node -p tcp --dport 2888 -j ACCEPT
  sudo iptables -A INPUT -s $node -p tcp --dport 3888 -j ACCEPT
done

# Drop all other ZooKeeper traffic
sudo iptables -A INPUT -p tcp --dport 2181 -j DROP
sudo iptables -A INPUT -p tcp --dport 2888 -j DROP
sudo iptables -A INPUT -p tcp --dport 3888 -j DROP
```

## Verifying ZooKeeper Binding

```bash
# Check ZooKeeper is listening on the right address
ss -tlnp | grep 2181
# Expected: 10.0.0.10:2181

# Test ZooKeeper is responding
echo ruok | nc 10.0.0.10 2181
# Expected: imok

# Get more detail
echo stat | nc 10.0.0.10 2181
# Shows: Mode (leader/follower/observer), connections, latency

# List Kafka brokers registered in ZooKeeper
echo "ls /kafka/brokers/ids" | /opt/zookeeper/bin/zkCli.sh -server 10.0.0.10:2181

# Check ZooKeeper quorum health
/opt/zookeeper/bin/zkServer.sh status
# Should show: Mode: leader (or follower)
```

## ZooKeeper systemd Service

```ini
# /etc/systemd/system/zookeeper.service
[Unit]
Description=Apache ZooKeeper
After=network.target

[Service]
Type=forking
User=zookeeper
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
ExecStart=/opt/zookeeper/bin/zkServer.sh start
ExecStop=/opt/zookeeper/bin/zkServer.sh stop
ExecReload=/opt/zookeeper/bin/zkServer.sh restart
PIDFile=/var/lib/zookeeper/zookeeper_server.pid

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now zookeeper
```

## Note on KRaft Mode

Starting with Kafka 3.3, KRaft (Kafka Raft) mode replaces ZooKeeper:

```properties
# /opt/kafka/config/kraft/server.properties

# KRaft mode — no ZooKeeper needed
process.roles=broker,controller
node.id=1

# Controller quorum uses Kafka ports (no ZooKeeper)
controller.quorum.voters=1@10.0.0.20:9093,2@10.0.0.21:9093,3@10.0.0.22:9093

listeners=PLAINTEXT://10.0.0.20:9092,CONTROLLER://10.0.0.20:9093
advertised.listeners=PLAINTEXT://10.0.0.20:9092
```

For new Kafka deployments, KRaft mode is preferred over ZooKeeper.

## Conclusion

ZooKeeper binds to a specific IPv4 address using `clientPortAddress` in `zoo.cfg`. For a 3-node ensemble, configure `server.1/2/3` entries with the IP addresses of each node, set unique `myid` files, and open ports 2181 (client), 2888 (peer), and 3888 (election) between ensemble members. Point Kafka to ZooKeeper via `zookeeper.connect` using IP addresses. For new deployments, consider KRaft mode which eliminates ZooKeeper entirely.
