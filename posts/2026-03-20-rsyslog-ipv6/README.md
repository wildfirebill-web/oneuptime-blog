# How to Configure rsyslog for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, rsyslog, Syslog, Linux, Log Management

Description: Configure rsyslog to accept log messages from IPv6 sources, forward logs to IPv6 servers, and use RainerScript to filter and route IPv6 log traffic.

## Introduction

rsyslog is the default syslog daemon on many Linux distributions including RHEL, Ubuntu, and Debian. It supports IPv6 input/output through its `imudp`, `imtcp`, and `omfwd` modules. This guide covers module configuration, IPv6 forwarding, and RainerScript-based filtering.

## Step 1: Enable IPv6 UDP and TCP Input

```ini
# /etc/rsyslog.conf or /etc/rsyslog.d/10-ipv6.conf

# Load input modules
module(load="imudp")
module(load="imtcp")

# UDP input — listen on all IPv6 interfaces
input(type="imudp"
      address="::1"
      port="514"
      name="udp-loopback")

# UDP input — all interfaces (IPv4 and IPv6)
input(type="imudp"
      port="5140"
      name="udp-all")

# TCP input — all IPv6 interfaces
input(type="imtcp"
      address="::"
      port="5140"
      name="tcp-ipv6")
```

## Step 2: Forward Logs to IPv6 Destinations

```ini
# Forward all logs to a remote server over IPv6 using UDP
*.* action(type="omfwd"
           Target="2001:db8::20"
           Port="514"
           Protocol="udp")

# Forward to remote server over IPv6 TCP with retries
*.* action(type="omfwd"
           Target="2001:db8::20"
           Port="514"
           Protocol="tcp"
           action.resumeRetryCount="-1"
           queue.type="linkedList"
           queue.size="10000"
           queue.filename="remote_ipv6_queue"
           queue.saveOnShutdown="on")

# Forward to Elasticsearch using omelasticsearch
module(load="omelasticsearch")
*.* action(type="omelasticsearch"
           server="[2001:db8::10]"
           serverport="9200"
           template="plain-syslog"
           searchIndex="syslog-%$year%.%$month%.%$day%")
```

## Step 3: Define Custom Templates

```ini
# JSON template for structured logging
template(name="ipv6-json" type="list") {
    constant(value="{")
    constant(value="\"timestamp\":\"")   property(name="timereported" dateFormat="rfc3339")
    constant(value="\",\"host\":\"")     property(name="hostname")
    constant(value="\",\"program\":\"")  property(name="programname")
    constant(value="\",\"message\":\"")  property(name="msg" format="json")
    constant(value="\",\"fromhost\":\"") property(name="fromhost-ip")
    constant(value="\"}\n")
}

# Use template for file output
*.* action(type="omfile"
           file="/var/log/ipv6-syslog.json"
           template="ipv6-json")
```

## Step 4: Filter by IPv6 Source Address

```ini
# RainerScript: route messages from specific IPv6 subnet
if ($fromhost-ip startswith "2001:db8:") then {
    action(type="omfile" file="/var/log/remote/2001-db8.log")
    stop
}

# Route link-local IPv6 sources
if ($fromhost-ip startswith "fe80:") then {
    action(type="omfile" file="/var/log/remote/link-local.log")
    stop
}

# Discard loopback
if ($fromhost-ip == "::1") then {
    stop
}

# Route all IPv6 sources (contains colon) to dedicated log
if ($fromhost-ip contains ":") then {
    action(type="omfile" file="/var/log/remote/ipv6-all.log"
           template="ipv6-json")
}
```

## Step 5: IPv6 Source with TLS

```ini
# Load TLS-capable TCP input module
module(load="imtcp"
       StreamDriver.Name="gtls"
       StreamDriver.Mode="1"
       StreamDriver.AuthMode="anon")

input(type="imtcp"
      port="6514"
      address="::"
      name="tls-ipv6")

# TLS forwarding to IPv6 server
global(
  DefaultNetStreamDriverCAFile="/etc/ssl/ca.pem"
  DefaultNetStreamDriverCertFile="/etc/ssl/rsyslog.pem"
  DefaultNetStreamDriverKeyFile="/etc/ssl/rsyslog.key"
)

*.* action(type="omfwd"
           Target="2001:db8::20"
           Port="6514"
           Protocol="tcp"
           StreamDriver="gtls"
           StreamDriverMode="1"
           StreamDriverAuthMode="anon")
```

## Step 6: Verify and Test

```bash
# Test rsyslog config syntax
rsyslogd -N1

# Send test message to IPv6 UDP listener
logger --server ::1 --port 5140 --ipv6 "Test IPv6 rsyslog message"

# Monitor incoming messages
tail -f /var/log/remote/ipv6-all.log

# Check rsyslog statistics
kill -USR1 $(pidof rsyslogd)
# Stats appear in /var/log/syslog or configured stats file
```

## Conclusion

rsyslog supports IPv6 through its `imudp` and `imtcp` modules by specifying `address="::"` for all-interface binding, and `omfwd` for forwarding to IPv6 destinations. RainerScript's `$fromhost-ip` property enables subnet-based routing using `startswith` or `contains` operators. For production deployments, combine IPv6 TCP forwarding with TLS using the `gtls` StreamDriver to secure log transport between rsyslog instances.
