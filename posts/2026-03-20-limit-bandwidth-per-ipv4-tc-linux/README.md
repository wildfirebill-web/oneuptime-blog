# How to Limit Bandwidth per IPv4 Address Using tc on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: tc, IPv4, Linux, Bandwidth, QoS, Per-IP

Description: Apply per-IP bandwidth limits on Linux using tc HTB with u32 filters or hash tables to control how much bandwidth each IPv4 address can use.

Limiting bandwidth per IPv4 address is useful for ISPs, shared hosting environments, or any situation where you want to prevent a single host from consuming all available bandwidth.

## Method 1: Explicit Per-IP Classes with u32 Filters

For a small number of IPs, create an HTB class for each and filter by source/destination IP:

```bash
# Create root HTB qdisc

sudo tc qdisc add dev eth0 root handle 1: htb default 999

# Root class: 1 Gbps total
sudo tc class add dev eth0 parent 1: classid 1:1 htb rate 1000mbit

# Catch-all class for unmatched traffic
sudo tc class add dev eth0 parent 1:1 classid 1:999 htb rate 100mbit ceil 1000mbit

# Per-IP classes - 10 Mbps limit for each IP
sudo tc class add dev eth0 parent 1:1 classid 1:100 htb rate 10mbit ceil 10mbit
sudo tc class add dev eth0 parent 1:1 classid 1:101 htb rate 10mbit ceil 10mbit
sudo tc class add dev eth0 parent 1:1 classid 1:102 htb rate 10mbit ceil 10mbit

# Filter: match source IP to its class
sudo tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
  match ip src 192.168.1.10/32 flowid 1:100

sudo tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
  match ip src 192.168.1.11/32 flowid 1:101

sudo tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 \
  match ip src 192.168.1.12/32 flowid 1:102
```

## Method 2: Hash Table Filters for Many IPs

For large numbers of IPs, use a hash table to avoid linear filter scanning:

```bash
# Create root HTB
sudo tc qdisc add dev eth0 root handle 1: htb default 30

# Per-IP limit class template (reused via hash)
sudo tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit
sudo tc class add dev eth0 parent 1:1 classid 1:10 htb rate 5mbit ceil 5mbit

# Create a hash filter table
sudo tc filter add dev eth0 parent 1: prio 5 handle 1: protocol ip u32 divisor 256

# Add an entry in the hash table for a specific IP
# The hash key is derived from the last octet of the IP
sudo tc filter add dev eth0 parent 1: prio 5 protocol ip u32 \
  ht 1:0a: \
  match ip dst 10.0.0.10/32 \
  flowid 1:10
```

## Method 3: Using iptables MARK + tc

Mark packets in iptables, then filter by mark in tc - this is more maintainable:

```bash
# Mark packets from specific IPs in iptables
sudo iptables -t mangle -A POSTROUTING -s 192.168.1.10 -j MARK --set-mark 10
sudo iptables -t mangle -A POSTROUTING -s 192.168.1.11 -j MARK --set-mark 11

# In tc, filter by the mark
sudo tc filter add dev eth0 protocol ip parent 1:0 handle 10 fw flowid 1:100
sudo tc filter add dev eth0 protocol ip parent 1:0 handle 11 fw flowid 1:101
```

## Automation Script: Apply Limits for All IPs in a Subnet

```bash
#!/bin/bash
# apply-per-ip-limits.sh
# Limits every IP in 192.168.1.0/24 to 5 Mbps

IFACE=eth0
RATE=5mbit

# Clean up
tc qdisc del dev $IFACE root 2>/dev/null

# Root HTB
tc qdisc add dev $IFACE root handle 1: htb default 999
tc class add dev $IFACE parent 1: classid 1:1 htb rate 1000mbit
tc class add dev $IFACE parent 1:1 classid 1:999 htb rate 100mbit

# Add a class and filter for each host IP
for i in $(seq 1 254); do
    CID=$((100 + i))
    tc class add dev $IFACE parent 1:1 classid "1:${CID}" htb rate $RATE ceil $RATE
    tc filter add dev $IFACE protocol ip parent 1:0 prio 1 u32 \
        match ip src "192.168.1.${i}/32" flowid "1:${CID}"
done

echo "Per-IP bandwidth limits applied"
```

This approach scales to hundreds of IPs and allows dynamic updates without full reconfiguration.
