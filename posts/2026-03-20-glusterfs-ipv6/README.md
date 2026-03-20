# How to Configure GlusterFS with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, GlusterFS, Distributed Storage, Network Storage, Linux

Description: Configure GlusterFS distributed file system to use IPv6 for inter-node communication and client mounts, including peer probing with IPv6 addresses and volume creation.

## Introduction

GlusterFS is a distributed file system that supports IPv6 for both peer-to-peer communication between storage nodes and client mounts. GlusterFS uses hostnames or IP addresses for peer identification, and IPv6 addresses must be used consistently throughout the configuration. The native GlusterFS client (`glusterfs`) and FUSE mount both support IPv6 server addresses.

## Prerequisites

```bash
# Install GlusterFS server on all nodes
apt-get install -y glusterfs-server    # Debian/Ubuntu
dnf install -y glusterfs-server        # RHEL/CentOS

# Enable and start GlusterFS daemon
systemctl enable --now glusterd

# Verify GlusterFS is listening on IPv6
ss -tlnp | grep glusterd
# Should show [::]:24007 (GlusterFS management port)
```

## Configure /etc/hosts for IPv6 GlusterFS Nodes

```bash
# /etc/hosts — on all GlusterFS nodes
# Use hostnames consistently across all nodes

2001:db8::10    gluster1 gluster1.example.com
2001:db8::11    gluster2 gluster2.example.com
2001:db8::12    gluster3 gluster3.example.com
```

## Peer Probing over IPv6

```bash
# From gluster1, probe other nodes using IPv6 addresses or hostnames
gluster peer probe gluster2
gluster peer probe gluster3

# Verify peer status
gluster peer status
# Expected:
# Number of Peers: 2
# Hostname: gluster2 (2001:db8::11)
# State: Peer in Cluster (Connected)
# Hostname: gluster3 (2001:db8::12)
# State: Peer in Cluster (Connected)
```

## Create a GlusterFS Volume with IPv6 Bricks

```bash
# Create distributed-replicated volume with IPv6 node addresses
gluster volume create myvol replica 3 \
    gluster1:/data/brick1 \
    gluster2:/data/brick1 \
    gluster3:/data/brick1

# Start the volume
gluster volume start myvol

# Verify volume info
gluster volume info myvol
# Expected:
# Volume Name: myvol
# Type: Replicate
# Volume ID: ...
# Status: Started
# Bricks:
# Brick1: gluster1:/data/brick1
# Brick2: gluster2:/data/brick1
# Brick3: gluster3:/data/brick1
```

## Mount GlusterFS Volume over IPv6 (Native Client)

```bash
# Mount using IPv6 server address directly
# The native GlusterFS client resolves hostnames to IPv6
mount -t glusterfs gluster1:/myvol /mnt/glusterfs

# Or explicitly using IPv6 address (requires DNS or /etc/hosts)
mount -t glusterfs [2001:db8::10]:/myvol /mnt/glusterfs

# Mount with options
mount -t glusterfs \
    -o log-level=WARNING,log-file=/var/log/gluster-client.log \
    gluster1:/myvol /mnt/glusterfs

# /etc/fstab entry
gluster1:/myvol   /mnt/glusterfs   glusterfs   defaults,_netdev   0   0
```

## GlusterFS Volume Options for IPv6

```bash
# Set volume transport to TCP (default, supports IPv6)
gluster volume set myvol transport.address-family inet6

# Enable auth for IPv6 client addresses
gluster volume set myvol auth.allow 2001:db8:clients::/48

# Check transport type
gluster volume info myvol | grep Transport
# Transport-type: tcp

# Enable glusterfs-server to listen on specific IPv6 address
# Edit /etc/glusterfs/glusterd.vol if needed
# Or set via environment: GLUSTERD_OPTIONS="--bind-address 2001:db8::10"
```

## Firewall Rules for GlusterFS over IPv6

```bash
# GlusterFS management port
ip6tables -A INPUT -p tcp --dport 24007 -s 2001:db8::/32 -j ACCEPT

# GlusterFS brick ports (24009 + one port per brick)
ip6tables -A INPUT -p tcp --dport 24008:24107 -s 2001:db8::/32 -j ACCEPT

# RDMA transport (if used)
# ip6tables -A INPUT -p tcp --dport 24011 -s 2001:db8::/32 -j ACCEPT

ip6tables-save > /etc/ip6tables/rules.v6
```

## Monitor GlusterFS over IPv6

```bash
# Check volume healing status
gluster volume heal myvol info

# Monitor volume status
gluster volume status myvol

# Check peer connections
gluster pool list

# Verify brick processes are running and using IPv6
ss -tlnp | grep glusterfs
# Should show brick processes listening on IPv6
```

## Troubleshooting GlusterFS IPv6 Issues

```bash
# Peer probe fails — check if glusterd is reachable
ping6 gluster2
telnet -6 gluster2 24007

# Volume mount fails — check gluster volume status
gluster volume status myvol

# Brick offline — check GlusterFS logs
tail -f /var/log/glusterfs/glusterd.log | grep -i error

# Check if transport is actually using IPv6
gluster volume get myvol transport.address-family
```

## Conclusion

GlusterFS supports IPv6 through standard TCP transport, which is IPv6-capable when IPv6 is configured on the network interfaces. The key requirement is consistent use of hostnames (resolved via `/etc/hosts` or DNS) or IPv6 addresses across all peer probe commands, volume creation, and client mounts. The `auth.allow` option accepts IPv6 CIDR notation for restricting client access. Firewall rules must allow the management port (24007) and brick ports (24008+) from the GlusterFS node CIDRs. Mount clients using `glusterfs` type with either hostnames resolving to IPv6 or direct IPv6 addresses in bracket notation.
