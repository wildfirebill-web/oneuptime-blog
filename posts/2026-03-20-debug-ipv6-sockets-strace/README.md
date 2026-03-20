# How to Debug IPv6 Socket Issues with strace and ltrace

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, strace, ltrace, Debugging, Socket Programming, Linux, Networking

Description: Use strace and ltrace to debug IPv6 socket issues in Linux applications by tracing system calls and library functions related to address families and socket operations.

## Introduction

`strace` traces system calls made by a process, while `ltrace` traces library function calls. Both tools are invaluable for debugging IPv6 socket issues when you need to see exactly what address family is being used, what addresses are being bound to, and where connections are failing.

## Basic strace for Socket Calls

```bash
# Trace all socket-related system calls
strace -e trace=network ./your-app

# Common socket syscalls to trace
strace -e trace=socket,bind,connect,accept,sendto,recvfrom ./your-app

# Also useful: getaddrinfo goes through name resolution
strace -e trace=network,read,write ./your-app 2>&1 | head -50
```

## Interpreting strace Socket Output

```bash
# Run your server and trace its socket calls
strace -p $(pgrep your-app) -e trace=socket,bind,listen 2>&1

# Example output - IPv4 only (bad):
# socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 5
# bind(5, {sa_family=AF_INET, sin_port=htons(8080), sin_addr=inet_addr("0.0.0.0")}, 16) = 0

# Example output - IPv6 (good):
# socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP) = 5
# setsockopt(5, SOL_IPV6, IPV6_V6ONLY, [0], 4) = 0
# bind(5, {sa_family=AF_INET6, sin6_port=htons(8080), sin6_addr=::}, 28) = 0
# listen(5, 5) = 0
```

## Tracing getaddrinfo Resolution

```bash
# Trace getaddrinfo and connect calls
strace -e trace=socket,connect,sendto -f ./your-client 2>&1 | grep -E "socket|connect|AF_"

# Output for IPv4-only client connecting to a dual-stack host:
# socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
# connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("93.184.216.34")}, 16) = 0
# (Only IPv4 tried - this is the bug if IPv6 should be used)

# Output for IPv6-capable client:
# socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP) = 3
# connect(3, {sa_family=AF_INET6, sin6_port=htons(80), sin6_flowinfo=0, inet_pton(AF_INET6, "2001:500:88:200::10", &sin6_addr), sin6_scope_id=0}, 28) = 0
```

## Detailed strace for IPv6 Issues

```bash
# Full trace with timestamps and output
strace -tt -T -e trace=network -o /tmp/strace.log ./your-app

# Find the socket family being used
grep "AF_INET" /tmp/strace.log

# Check for bind failures
grep "bind.*EADDRNOTAVAIL\|bind.*EINVAL\|bind.*= -1" /tmp/strace.log

# Check for connect failures
grep "connect.*= -1\|ECONNREFUSED\|ENETUNREACH\|EHOSTUNREACH" /tmp/strace.log
```

## Using ltrace for Library Functions

```bash
# Trace DNS resolution library calls
ltrace -e getaddrinfo ./your-app 2>&1 | head -20

# Example ltrace output:
# getaddrinfo("example.com", "80", { ai_family=AF_UNSPEC, ... }, &result) = 0
# -- or if IPv6 not working --
# getaddrinfo("example.com", "80", { ai_family=AF_INET, ... }, &result) = 0
# (AF_INET hardcoded = bug)

# Trace specific library calls related to IPv6
ltrace -e inet_pton+inet_ntop+getaddrinfo+getnameinfo ./your-app 2>&1
```

## Diagnosing "Network is unreachable" for IPv6

```bash
# Trace connect() for ENETUNREACH errors
strace -e trace=connect ./your-client 2>&1 | grep -E "connect|ENETUNREACH"

# Example showing failure for link-local without scope ID:
# connect(3, {sa_family=AF_INET6, sin6_port=htons(8080),
#             sin6_addr=fe80::1, sin6_scope_id=0}, 28) = -1 ENETUNREACH
# Fix: set sin6_scope_id = if_nametoindex("eth0")

# EINVAL for scope ID issues:
# connect(3, {sa_family=AF_INET6, sin6_port=htons(8080),
#             sin6_addr=fe80::1, sin6_scope_id=0}, 28) = -1 EINVAL
```

## Finding Hard-Coded IPv4 in Binaries

```bash
# Strace to find AF_INET (IPv4-only) socket creation
strace -e trace=socket ./your-app 2>&1 | grep "AF_INET[^6]"
# Any AF_INET without AF_INET6 indicates IPv4-only socket

# Check setsockopt for IPV6_V6ONLY
strace -e trace=setsockopt ./your-app 2>&1 | grep IPV6
# Should see: setsockopt(fd, SOL_IPV6, IPV6_V6ONLY, [0], 4) = 0 for dual-stack
```

## strace Attach to Running Process

```bash
# Attach to a running server to watch IPv6 accept calls
sudo strace -p $(pgrep -n nginx) -e trace=accept4,accept 2>&1 | \
    grep "AF_INET6\|AF_INET"

# Watch for IPv6 connections in real-time
sudo strace -p $(pgrep -n my-server) -e trace=accept 2>&1
# Output for IPv6 client:
# accept(5, {sa_family=AF_INET6, sin6_port=htons(54321), sin6_addr=2001:db8::1, ...}, [28]) = 6
```

## Practical Debug Script

```bash
#!/bin/bash
# debug-ipv6-sockets.sh

APP="$1"
if [ -z "$APP" ]; then
    echo "Usage: $0 <app-name-or-pid>"
    exit 1
fi

echo "=== IPv6 Socket Debug for: $APP ==="

# Get PID
if [[ "$APP" =~ ^[0-9]+$ ]]; then
    PID=$APP
else
    PID=$(pgrep -n "$APP")
fi

if [ -z "$PID" ]; then
    echo "Process not found, running strace on launch..."
    strace -e trace=socket,bind,connect,setsockopt -f ./$APP 2>&1 | \
        grep -E "AF_INET|IPV6|bind|connect" | head -30
else
    echo "Attaching to PID $PID..."
    timeout 10 strace -p $PID -e trace=socket,accept,connect 2>&1 | \
        grep -E "AF_INET|IPV6" | head -30
fi
```

## Conclusion

`strace` with `-e trace=network` is the most effective tool for debugging IPv6 socket issues. Look for `AF_INET` instead of `AF_INET6` in `socket()` calls to find IPv4-only code paths, check `bind()` calls for the address being bound to, and examine `connect()` calls for `ENETUNREACH` errors that indicate missing routes or scope IDs. The combination of strace and `ss -tlnp` provides complete visibility into IPv6 socket behavior.
