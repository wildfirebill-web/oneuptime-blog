# How to Configure IPv6 for Cross-Data-Center Replication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Replication, Data Center, Database, Storage

Description: Configure IPv6 for cross-data-center database and storage replication, including addressing, firewall rules, and latency tuning.

## Why IPv6 Simplifies Cross-DC Replication

Cross-data-center replication over IPv6 eliminates NAT traversal issues and simplifies firewall rules. Each replication endpoint has a globally unique, routable IPv6 address, making peer configuration straightforward.

## Address Planning for Replication Traffic

Dedicate a specific subnet for replication traffic to allow targeted QoS and firewall policies:

```
DC1 Replication Subnet: 2001:db8:1:repl::/64  (hex: 2001:db8:1:7265::/64)
DC2 Replication Subnet: 2001:db8:2:repl::/64

DC1 DB Primary:  2001:db8:1:7265::10
DC2 DB Replica:  2001:db8:2:7265::10
```

## PostgreSQL Streaming Replication over IPv6

Configure PostgreSQL primary to accept replication connections over IPv6:

```
# postgresql.conf on Primary (DC1)
listen_addresses = '*'
wal_level = replica
max_wal_senders = 5

# pg_hba.conf - allow replica from DC2 replication subnet
host  replication  replicator  2001:db8:2:7265::/64  scram-sha-256
```

Connect the standby to the primary using an IPv6 connection string:

```
# recovery.conf / postgresql.conf on Replica (DC2)
primary_conninfo = 'host=2001:db8:1:7265::10 port=5432 user=replicator password=secret'
```

## MySQL Group Replication over IPv6

MySQL Group Replication requires configuring the communication stack with IPv6 addresses:

```sql
-- On each node, set the local address
SET GLOBAL group_replication_local_address = '[2001:db8:1:7265::10]:33061';

-- Set seeds pointing to DC2 replica
SET GLOBAL group_replication_group_seeds = '[2001:db8:2:7265::10]:33061';
```

## Firewall Rules for Replication Traffic

Allow replication ports only between the designated replication subnets:

```bash
# Allow PostgreSQL replication from DC2 to DC1
ip6tables -A INPUT -s 2001:db8:2:7265::/64 -d 2001:db8:1:7265::/64 \
  -p tcp --dport 5432 -j ACCEPT

# Drop replication port access from all other sources
ip6tables -A INPUT -p tcp --dport 5432 -j DROP
```

## Storage Replication: Ceph over IPv6

Configure Ceph monitors and OSDs to bind to IPv6 addresses:

```
# ceph.conf
[global]
ms_bind_ipv6 = true
ms_bind_ipv4 = false

[mon.dc1-mon1]
mon_host = [2001:db8:1:7265::1]:6789

[mon.dc2-mon1]
mon_host = [2001:db8:2:7265::1]:6789
```

## Latency and Bandwidth Tuning

Cross-DC replication is latency-sensitive. Tune TCP buffer sizes for high-latency links:

```bash
# Increase TCP send/receive buffers for replication connections
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv6.conf.all.use_oif_addrs_only=1
```

## Monitoring Replication Lag

Use Prometheus and node_exporter to track replication lag. Pair with alerting in OneUptime to notify when replication falls behind a threshold.

## Conclusion

Configuring cross-data-center replication over IPv6 simplifies addressing and security policy. Use dedicated replication subnets, explicit firewall rules, and TCP buffer tuning to achieve reliable, low-latency replication across sites.
