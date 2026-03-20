# How to Perform IPv6 Port Scanning Techniques

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Port Scanning, nmap, Security Testing, Network Reconnaissance, TCP

Description: A guide to IPv6 port scanning techniques using nmap and other tools, covering SYN scans, UDP scans, version detection, and evasion techniques over IPv6.

IPv6 port scanning uses the same TCP/UDP scanning techniques as IPv4, but requires IPv6-capable tools and specific flags. The primary challenge with IPv6 is host discovery (addressed in separate guides) — once you have a target IPv6 address, port scanning proceeds similarly to IPv4.

## Basic IPv6 Port Scanning with nmap

```bash
# Full port scan of a single IPv6 host
nmap -6 -p- 2001:db8::target

# Fast scan (top 100 ports)
nmap -6 -F 2001:db8::target

# Specific port ranges
nmap -6 -p 22,80,443,8080,3306 2001:db8::target

# All 65535 ports
nmap -6 -p 0-65535 2001:db8::target
```

## TCP Scanning Techniques

```bash
# SYN scan (stealth, requires root)
sudo nmap -6 -sS 2001:db8::target

# TCP Connect scan (no root required, more detectable)
nmap -6 -sT 2001:db8::target

# FIN scan (sends FIN without SYN — evades some filters)
sudo nmap -6 -sF 2001:db8::target

# NULL scan (no flags set)
sudo nmap -6 -sN 2001:db8::target

# XMAS scan (FIN, PSH, URG flags)
sudo nmap -6 -sX 2001:db8::target

# ACK scan (used to map firewall rules)
sudo nmap -6 -sA 2001:db8::target
```

## UDP Scanning

```bash
# UDP scan (slow, requires root)
sudo nmap -6 -sU 2001:db8::target

# UDP scan with version detection
sudo nmap -6 -sU -sV 2001:db8::target

# Common UDP ports
sudo nmap -6 -sU -p 53,123,161,500,4500 2001:db8::target

# Combine TCP and UDP
sudo nmap -6 -sS -sU -p T:80,443 -p U:53,161 2001:db8::target
```

## Service and Version Detection

```bash
# Detect service versions
nmap -6 -sV 2001:db8::target

# OS fingerprinting (requires root)
sudo nmap -6 -O 2001:db8::target

# Aggressive scan (OS, version, scripts, traceroute)
sudo nmap -6 -A 2001:db8::target

# NSE scripts for specific services
nmap -6 -p 80,443 --script http-headers 2001:db8::target
nmap -6 -p 22 --script ssh-hostkey 2001:db8::target
nmap -6 -p 25 --script smtp-commands 2001:db8::target
```

## Scan Performance Tuning

```bash
# Timing templates (T1=sneaky, T2=polite, T3=normal, T4=aggressive, T5=insane)
nmap -6 -T4 2001:db8::target        # Aggressive
nmap -6 -T2 2001:db8::target        # Slow (evasion)

# Set minimum/maximum parallelism
nmap -6 --min-parallelism 10 --max-parallelism 100 2001:db8::target

# Set host timeout
nmap -6 --host-timeout 30s 2001:db8::target

# Reduce retries
nmap -6 --max-retries 2 2001:db8::target
```

## Firewall Evasion Techniques

```bash
# Fragment packets (may bypass naive packet inspection)
sudo nmap -6 -f 2001:db8::target

# Use decoys (mix real scan with fake source IPs)
sudo nmap -6 -D 2001:db8::decoy1,2001:db8::decoy2,ME 2001:db8::target

# Spoof source address (for testing firewall rules)
sudo nmap -6 -S 2001:db8::spoofed-src -e eth0 2001:db8::target

# Custom TTL/Hop Limit
nmap -6 --ttl 32 2001:db8::target

# Randomize scan order
nmap -6 --randomize-hosts -iL targets.txt
```

## Alternative Tools for IPv6 Port Scanning

```bash
# masscan (very fast, IPv6 support)
sudo masscan -6 --ports 80,443,22 2001:db8::target

# zmap (research scanner, IPv6 support in zmap6)
sudo zmap6 -p 443 -i eth0 2001:db8::/48

# unicornscan
sudo unicornscan -mT "[2001:db8::target]":1-1024
```

## Scanning a List of IPv6 Targets

```bash
# Input from file
nmap -6 -sV -iL ipv6-targets.txt -oA scan-results

# targets.txt format:
# 2001:db8::10
# 2001:db8::11
# 2001:db8::gateway
```

Always obtain written authorization before port scanning any system. IPv6 doesn't provide anonymity — scans are logged by target hosts just as IPv4 scans are.
