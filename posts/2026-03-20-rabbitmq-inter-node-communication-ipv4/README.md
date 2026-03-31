# How to Configure RabbitMQ Inter-Node Communication on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RabbitMQ, IPv4, Inter-Node, Erlang Distribution, Clustering, Network

Description: Configure RabbitMQ inter-node communication ports on IPv4, including Erlang distribution (25672) and EPMD (4369), for reliable cluster operation.

## Introduction

RabbitMQ cluster nodes communicate via the Erlang Distribution Protocol. Two ports are involved: EPMD (Erlang Port Mapper Daemon) on port 4369, which helps nodes find each other, and the Erlang distribution port (default 25672), which carries actual cluster traffic.

## Port Overview

| Port | Service | Purpose |
|---|---|---|
| 4369 | EPMD | Node discovery within cluster |
| 25672 | Erlang distribution | Cluster data and messages |
| 5672 | AMQP | Client connections |
| 15672 | Management | HTTP API / web UI |

## Configuring the Distribution Port

```bash
# /etc/rabbitmq/rabbitmq.conf

# Restrict inter-node communication to specific IP

distribution.listener.interface = 10.0.0.1    # Node-specific IP

# Or configure via kernel inet_dist_listen_min/max in advanced.config
```

```erlang
%% /etc/rabbitmq/advanced.config
%% Fix Erlang distribution port range (optional but helps with firewalls)
[
  {kernel, [
    {inet_dist_listen_min, 25672},
    {inet_dist_listen_max, 25672}
  ]}
].
```

## Node Name and IP Address

```bash
# RabbitMQ node names use either hostname or IP
# Default: rabbit@hostname

# To use IP-based node names (less common):
# Set in /etc/rabbitmq/rabbitmq-env.conf
NODENAME=rabbit@10.0.0.1

# This affects all rabbit* commands:
sudo rabbitmqctl -n rabbit@10.0.0.1 status
```

## Firewall Rules for Inter-Node Traffic

```bash
#!/bin/bash
# Configure firewall for RabbitMQ cluster inter-node communication

NODES=("10.0.0.1" "10.0.0.2" "10.0.0.3")

for node in "${NODES[@]}"; do
  # EPMD - node discovery
  sudo iptables -A INPUT -p tcp --dport 4369 -s "$node" -j ACCEPT

  # Erlang distribution port
  sudo iptables -A INPUT -p tcp --dport 25672 -s "$node" -j ACCEPT
done

# App clients: AMQP only, no inter-node ports
sudo iptables -A INPUT -p tcp --dport 5672 -s 10.0.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 5672 -j DROP

# Block EPMD and distribution from non-cluster IPs
sudo iptables -A INPUT -p tcp --dport 4369 -j DROP
sudo iptables -A INPUT -p tcp --dport 25672 -j DROP
```

## Troubleshooting Inter-Node Connectivity

```bash
# Test if EPMD can reach another node
epmd -names  # List registered Erlang nodes locally
ssh 10.0.0.2 "epmd -names"  # Check on remote node

# Test Erlang distribution port
nc -zv 10.0.0.2 25672

# Check cluster connectivity from RabbitMQ perspective
sudo rabbitmqctl node_health_check -n rabbit@node2

# Check if nodes see each other
sudo rabbitmqctl cluster_status | grep -A5 "Running Nodes"

# Check logs for distribution errors
sudo grep -i "connect\|distribution\|net_kernel" /var/log/rabbitmq/rabbit@*.log
```

## Conclusion

RabbitMQ inter-node communication uses EPMD (4369) and Erlang distribution (25672). Both must be open between all cluster nodes. Use `distribution.listener.interface` in `rabbitmq.conf` to bind the distribution listener to a specific IPv4. Block these ports from non-cluster networks-only AMQP clients need port 5672. Fix the distribution port range in `advanced.config` to make iptables rules predictable.
