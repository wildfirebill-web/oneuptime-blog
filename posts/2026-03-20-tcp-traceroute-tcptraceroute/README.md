# How to Run TCP Traceroute with tcptraceroute

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Traceroute, TCP, Linux, Networking, Firewall, Diagnostic

Description: Use tcptraceroute to trace network paths using TCP SYN packets instead of UDP or ICMP, bypassing firewalls that block traditional traceroute probes.

Standard traceroute often hits firewalls that block ICMP or high UDP ports, showing rows of asterisks. tcptraceroute uses TCP SYN packets on real service ports (80, 443, 22), allowing it to traverse the same path as actual application traffic.

## Why TCP Traceroute?

```text
Traditional traceroute problems:
  - Uses UDP to high ports (>33434) - often blocked by firewalls
  - ICMP mode requires root and is commonly filtered
  - Shows *** even when the path is working for TCP traffic

tcptraceroute advantage:
  - Uses TCP SYN to any port you specify (80, 443, 22)
  - Travels the same path as real application traffic
  - Sees through firewalls configured to allow HTTP/HTTPS
  - Identifies EXACTLY where TCP connectivity breaks
```

## Install tcptraceroute

```bash
# Debian/Ubuntu

sudo apt install tcptraceroute -y

# RHEL/CentOS
sudo yum install tcptraceroute -y

# macOS
brew install tcptraceroute
```

## Basic Usage

```bash
# TCP traceroute to port 80 (HTTP)
sudo tcptraceroute google.com 80

# TCP traceroute to port 443 (HTTPS)
sudo tcptraceroute google.com 443

# TCP traceroute to port 22 (SSH)
sudo tcptraceroute github.com 22

# Numeric output (skip DNS)
sudo tcptraceroute -n google.com 80
```

## Reading tcptraceroute Output

```bash
sudo tcptraceroute -n 8.8.8.8 80
# Selected device eth0, address 192.168.1.100, port 54321 for outgoing packets
# Tracing the path to 8.8.8.8 on TCP port 80, 30 hops max
#  1  192.168.1.1   1.3 ms  1.2 ms  1.1 ms
#  2  10.1.0.1      8.5 ms  8.3 ms  8.4 ms
#  3  * * *                          ← router doesn't respond to probes
#  4  8.8.8.8 [open]  12.8 ms  12.7 ms  12.9 ms
#              ^^^^^
#     [open] = destination accepted the SYN (port is open and reachable)
#     [closed] = destination reset the SYN (port closed but routed OK)
#     no response = TCP is being filtered or host is down
```

## Comparing Standard vs TCP Traceroute

```bash
# Standard traceroute (UDP) - many stars due to firewall
traceroute -n 8.8.8.8
#  1  192.168.1.1   1ms
#  2  * * *
#  3  * * *
#  4  * * *
# (can't see the path)

# TCP traceroute on port 80 - traverses same path as HTTP
sudo tcptraceroute -n 8.8.8.8 80
#  1  192.168.1.1   1ms
#  2  10.1.0.1      8ms
#  3  72.14.0.1    12ms
#  4  8.8.8.8 [open]  12ms
# (full path visible)
```

## Diagnose Web Application Connectivity

```bash
# Check if web server is reachable and which hop blocks it
sudo tcptraceroute -n app.example.com 443

# If it shows [open] at destination → web server is up and reachable
# If it shows stars at destination → port 443 is filtered at that point
# If it shows [closed] at destination → server is up but SSL not configured
```

## Using with traceroute -T (Alternative)

If tcptraceroute isn't available, traceroute itself supports TCP mode:

```bash
# Built-in TCP mode in traceroute
sudo traceroute -T -p 80 -n 8.8.8.8   # TCP SYN to port 80
sudo traceroute -T -p 443 -n 8.8.8.8  # TCP SYN to port 443

# traceroute TCP flags (add --sport to control source port)
sudo traceroute -T -p 22 --sport=54321 -n 192.168.1.1
```

## Troubleshoot "Timeout" Connections

```bash
# A web connection that hangs (not refused, not accepted) → firewall DROP
# Trace the TCP path:
sudo tcptraceroute -n problematic-server.com 443

# If last hop shows * * * before destination:
# → A firewall is silently dropping the TCP SYN
# → Get the last visible IP and check that device's ACL
```

TCP traceroute is an essential tool for diagnosing application-level connectivity - it tests the actual path your application traffic takes, not just whether ICMP is allowed.
