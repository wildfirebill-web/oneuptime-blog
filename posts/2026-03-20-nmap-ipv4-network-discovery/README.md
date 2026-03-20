# How to Use nmap for IPv4 Network Discovery

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: nmap, IPv4, Network Discovery, Security, Linux, Scanning

Description: Use nmap to discover live hosts, open ports, and services on IPv4 networks, from basic ping sweeps to comprehensive service version detection.

nmap is the standard tool for network discovery and security auditing. It can tell you which hosts are up, what ports they have open, what services are running, and even what operating systems they're running.

## Basic Host Discovery (Ping Sweep)

```bash
# Scan a single host (ping + port scan)

nmap 192.168.1.100

# Ping sweep - discover all live hosts in a subnet
nmap -sn 192.168.1.0/24

# Ping sweep on multiple networks
nmap -sn 192.168.1.0/24 192.168.2.0/24 10.0.0.0/24

# Output:
# Starting Nmap ...
# Nmap scan report for 192.168.1.1
# Host is up (0.0012s latency).
# Nmap scan report for 192.168.1.100
# Host is up (0.0034s latency).
# 12 hosts are up (53 down)
```

## Port Scanning

```bash
# Scan most common 1000 ports (default)
nmap 192.168.1.100

# Scan all 65535 ports
nmap -p- 192.168.1.100

# Scan specific ports
nmap -p 22,80,443,3306 192.168.1.100

# Scan a port range
nmap -p 1-1024 192.168.1.100

# Fast scan (top 100 ports)
nmap -F 192.168.1.100
```

## TCP SYN Scan (Most Common)

```bash
# SYN scan - fast, stealthy (doesn't complete handshake)
sudo nmap -sS 192.168.1.100

# TCP Connect scan (no root needed, completes handshake)
nmap -sT 192.168.1.100

# UDP scan (discovers UDP services like DNS, SNMP)
sudo nmap -sU 192.168.1.100

# Combined TCP + UDP scan
sudo nmap -sSU -p T:22,80,443,U:53,161 192.168.1.100
```

## Service and Version Detection

```bash
# Detect service versions (-sV)
nmap -sV 192.168.1.100

# Aggressive scan: version + OS + scripts + traceroute
sudo nmap -A 192.168.1.100

# Output:
# PORT    STATE SERVICE  VERSION
# 22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu
# 80/tcp  open  http     nginx 1.18.0
# 443/tcp open  ssl/http nginx 1.18.0
```

## OS Detection

```bash
# Detect operating system
sudo nmap -O 192.168.1.100

# Output:
# OS details: Linux 4.15 - 5.6
# Network Distance: 1 hop
```

## Scan an Entire Network with Output

```bash
# Scan subnet and save output in multiple formats
sudo nmap -sn -oA /tmp/discovery 192.168.1.0/24
# Creates: discovery.nmap, discovery.xml, discovery.gnmap

# Scan for open web ports across subnet
nmap -p 80,443,8080 --open 192.168.1.0/24

# Scan with timing template (T0=slowest, T5=fastest)
nmap -T4 -sS 192.168.1.0/24    # Aggressive, faster

# Exclude specific hosts from scan
nmap 192.168.1.0/24 --exclude 192.168.1.1,192.168.1.254
```

## Scripting Engine (NSE)

```bash
# Run default scripts
nmap -sC 192.168.1.100

# Run specific script
nmap --script=http-title 192.168.1.100

# Vulnerability scan (careful - can be noisy)
sudo nmap --script=vuln 192.168.1.100

# SSL/TLS configuration check
nmap --script=ssl-enum-ciphers -p 443 192.168.1.100
```

Always obtain permission before scanning networks you don't own - unauthorized nmap scans may violate computer fraud laws and network policies.
