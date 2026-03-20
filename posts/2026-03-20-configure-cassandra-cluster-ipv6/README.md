# How to Configure Apache Cassandra Cluster with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Cassandra, IPv6, Distributed Database, NoSQL, Cluster, Gossip Protocol

Description: Configure an Apache Cassandra cluster to use IPv6 addresses for gossip communication, native transport, and RPC, enabling NoSQL database clustering over IPv6 networks.

---

Apache Cassandra uses a gossip protocol for cluster communication. Configuring it for IPv6 requires updating `cassandra.yaml` with IPv6 listen and broadcast addresses for both gossip and native protocol communications.

## Cassandra IPv6 Key Configuration Parameters

```yaml
# Critical parameters in cassandra.yaml for IPv6:
# listen_address - What address Cassandra binds to
# broadcast_address - What address other nodes use to reach this node
# rpc_address - What address clients connect to
# broadcast_rpc_address - What clients are told to connect to
```

## Configuring cassandra.yaml for IPv6

```yaml
# /etc/cassandra/cassandra.yaml

# Cluster name
cluster_name: 'IPv6 Cluster'

# Listen on IPv6 address for inter-node communication
listen_address: 2001:db8::1
# NOT listen_interface when using explicit address

# Broadcast this IPv6 address to other nodes
broadcast_address: 2001:db8::1

# RPC (Thrift/Native) address for client connections
# Use :: for all interfaces including IPv6
rpc_address: ::
# Or specific IPv6 address:
# rpc_address: 2001:db8::1

# Broadcast RPC address to clients
broadcast_rpc_address: 2001:db8::1

# Seeds - other nodes to connect to for bootstrapping
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "2001:db8::1,2001:db8::2,2001:db8::3"

# Ports remain the same as IPv4
storage_port: 7000
ssl_storage_port: 7001
native_transport_port: 9042

# Enable IPv6 for JMX
# Add to jvm.options:
# -Djava.net.preferIPv6Addresses=true
```

## JVM Options for IPv6

```bash
# /etc/cassandra/jvm.options or jvm-server.options
# Force JVM to prefer IPv6 addresses
-Djava.net.preferIPv6Addresses=true

# Bind JMX to IPv6 loopback
-Dcom.sun.jndi.rmiURLParsing=legacy
```

## Starting Cassandra on IPv6

```bash
# Start Cassandra
sudo systemctl start cassandra

# Verify it's listening on IPv6
ss -tlnp | grep "9042\|7000"
# Should show [::]:9042 for native transport
# And [2001:db8::1]:7000 for storage

# Check Cassandra status
nodetool status
# Should show IPv6 address in the output

# Check gossip information
nodetool gossipinfo | head -20
```

## Connecting to Cassandra over IPv6

```bash
# Use cqlsh with IPv6 address
cqlsh 2001:db8::1

# Or with explicit port
cqlsh 2001:db8::1 9042

# Test connection
cqlsh 2001:db8::1 -e "SELECT cluster_name FROM system.local;"
```

```python
# Python driver connection over IPv6
from cassandra.cluster import Cluster
from cassandra.policies import DCAwareRoundRobinPolicy

# Connect to Cassandra cluster via IPv6
cluster = Cluster(
    contact_points=['2001:db8::1', '2001:db8::2', '2001:db8::3'],
    port=9042,
    load_balancing_policy=DCAwareRoundRobinPolicy(local_dc='datacenter1')
)

session = cluster.connect()
rows = session.execute("SELECT cluster_name FROM system.local")
for row in rows:
    print(f"Connected to cluster: {row.cluster_name}")

cluster.shutdown()
```

## Multi-Node Cluster Configuration

For a 3-node cluster, configure each node differently:

```bash
# Node 1: listen_address: 2001:db8::1
# Node 2: listen_address: 2001:db8::2
# Node 3: listen_address: 2001:db8::3

# All nodes share the same seeds:
# seeds: "2001:db8::1,2001:db8::2,2001:db8::3"

# Bootstrap node 2 and 3 AFTER node 1 is running
sudo systemctl start cassandra  # On node 2 after node 1 is up
```

## Firewall Rules for Cassandra IPv6

```bash
# Allow Cassandra ports for IPv6
sudo ip6tables -A INPUT -p tcp --dport 7000 -j ACCEPT   # Storage/gossip
sudo ip6tables -A INPUT -p tcp --dport 7001 -j ACCEPT   # SSL storage
sudo ip6tables -A INPUT -p tcp --dport 9042 -j ACCEPT   # Native transport
sudo ip6tables -A INPUT -p tcp --dport 7199 -j ACCEPT   # JMX

# Restrict to cluster subnet
sudo ip6tables -A INPUT -p tcp -s 2001:db8::/48 --dport 7000 -j ACCEPT

sudo ip6tables-save > /etc/ip6tables/rules.v6
```

Apache Cassandra's IPv6 support through configurable listen and broadcast addresses enables building geographically distributed NoSQL clusters on modern IPv6 infrastructure with the same scalability characteristics as IPv4 deployments.
