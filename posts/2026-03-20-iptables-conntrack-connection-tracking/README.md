# How to Use Connection Tracking (conntrack) with iptables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: iptables, Conntrack, Linux, Firewall, Stateful, Networking

Description: Use iptables connection tracking (conntrack) and the state module to create stateful firewall rules that track TCP session states and automatically handle return traffic.

Connection tracking is what makes iptables a stateful firewall. Without it, you'd need separate rules for both directions of every connection. With it, you write rules for new connections only - iptables automatically allows established and related packets back.

## How Connection Tracking Works

```text
Without conntrack:
  Need: allow outbound TCP port 80 AND allow inbound responses on ephemeral ports
  (impossible to write reliable rules for random return ports)

With conntrack:
  Track: "host 10.0.0.1 made a NEW connection to 8.8.8.8:80"
  Auto-allow: any ESTABLISHED packet back from 8.8.8.8 to 10.0.0.1 on that flow

States:
  NEW         - First packet of a new connection
  ESTABLISHED - Part of an existing connection
  RELATED     - Related to an existing connection (FTP data, ICMP errors)
  INVALID     - Doesn't match any known state (should be dropped)
```

## Basic Stateful Firewall

```bash
# Allow established and related connections (covers return traffic)

sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow new inbound connections to specific services
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT

# Allow all new outbound connections (from this server)
sudo iptables -A OUTPUT -m state --state NEW -j ACCEPT

# Drop everything else
sudo iptables -A INPUT -j DROP

# This is a complete stateful firewall in 6 rules
```

## Drop Invalid Packets

```bash
# Invalid packets don't belong to any tracked connection
# They should always be dropped
sudo iptables -I INPUT 1 -m state --state INVALID -j DROP
sudo iptables -I FORWARD 1 -m state --state INVALID -j DROP
sudo iptables -I OUTPUT 1 -m state --state INVALID -j DROP
```

## View the Connection Tracking Table

```bash
# Install conntrack tools
sudo apt install conntrack -y

# View all tracked connections
sudo conntrack -L

# View only TCP connections
sudo conntrack -L -p tcp

# View only ESTABLISHED connections
sudo conntrack -L | grep ESTABLISHED

# View specific IP's connections
sudo conntrack -L | grep 192.168.1.50

# Example output:
# tcp      6 86385 ESTABLISHED src=192.168.1.100 dst=8.8.8.8 sport=54321 dport=80
#          [ASSURED] mark=0 use=1
```

## Manage Connection Table

```bash
# Delete all entries for a specific IP (kick connections)
sudo conntrack -D -s 1.2.3.4

# Delete entries to a specific destination
sudo conntrack -D -d 10.0.0.50

# Flush entire connection table (use with caution)
sudo conntrack -F

# Get connection table statistics
sudo conntrack -S

# Monitor connection events in real time
sudo conntrack -E
```

## FTP and Other Multi-Connection Protocols

RELATED state handles "related" connections like FTP data channels:

```bash
# FTP uses two connections: control (port 21) and data (random port)
# conntrack has an FTP helper that marks data connections as RELATED

# Load FTP helper module
sudo modprobe nf_conntrack_ftp

# Allow FTP control
sudo iptables -A INPUT -p tcp --dport 21 -m state --state NEW -j ACCEPT

# Allow related FTP data connections (auto-tracked by helper)
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

## Tune Connection Tracking Limits

```bash
# Check current conntrack table utilization
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# Increase max for high-traffic servers
sudo sysctl -w net.netfilter.nf_conntrack_max=262144
```

Connection tracking is the foundation of stateful firewalling - it eliminates the need to write complex rules for return traffic and enables accurate matching based on TCP session state.
