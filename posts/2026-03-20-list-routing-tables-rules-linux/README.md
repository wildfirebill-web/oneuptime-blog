# How to List All Routing Tables and Rules on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Routing, iproute2, Policy Routing, Ip rule, Networking

Description: List all routing tables and policy routing rules on Linux using iproute2 commands to audit and understand how traffic is routed across multiple tables.

## Introduction

Linux uses multiple routing tables evaluated in priority order. The `ip rule` command shows the policy routing database (which table to use), while `ip route show table` shows routes in each table. Understanding these is essential for diagnosing complex routing issues.

## List All Routing Rules

```bash
# Show all ip rules (policy routing database)

ip rule list

# Sample output:
# 0:      from all lookup local
# 32766:  from all lookup main
# 32767:  from all lookup default
```

Rules are evaluated in ascending priority order (lower number = higher priority).

## List Routes in the Main Routing Table

```bash
# Default table used for most traffic
ip route show table main

# Shorthand (same as above)
ip route show
```

## List Routes in All Tables

```bash
# Show every route in every table
ip route show table all

# Filter by table number or name
ip route show table 100
ip route show table local
ip route show table default
```

## List Custom Tables Defined on the System

```bash
# Show registered routing table names and IDs
cat /etc/iproute2/rt_tables

# Default output:
# 255  local
# 254  main
# 253  default
# 0    unspec
# (+ any custom tables you've added)
```

## Check Which Table a Packet Would Use

```bash
# Trace which rule and table handles a packet from 10.0.0.5
ip rule show | head -20

# Test actual routing decision for a destination
ip route get 8.8.8.8
ip route get 8.8.8.8 from 10.0.0.5
```

## Count Routes Per Table

```bash
# Count routes in main table
ip route show table main | wc -l

# Count routes in local table
ip route show table local | wc -l

# Show route count for every non-empty table
for table in $(ip route show table all | grep -oP 'table \K\S+' | sort -u); do
    count=$(ip route show table $table 2>/dev/null | wc -l)
    echo "Table $table: $count routes"
done
```

## List Only Specific Route Types

```bash
# Show only blackhole routes
ip route show type blackhole

# Show only unreachable routes
ip route show type unreachable

# Show only unicast routes
ip route show type unicast

# Show only multicast routes
ip route show table local type multicast
```

## View Detailed Rule Information

```bash
# Show rules with verbose output
ip -detail rule list

# Show rules for IPv6
ip -6 rule list
```

## Conclusion

Use `ip rule list` to see the policy routing database and `ip route show table <name>` to inspect routes in each table. The main table handles most traffic; the local table handles local and broadcast addresses. For full visibility, `ip route show table all` dumps every installed route regardless of table.
