# How to List All Network Namespaces on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Namespaces, iproute2, Networking, Containers, System Administration

Description: List and inspect all network namespaces on a Linux system using ip netns list, /var/run/netns, and lsns to understand the current namespace layout.

## Introduction

Listing network namespaces helps you understand what isolated network environments exist on a system - whether created manually, by containers, or by network tools. Linux provides several ways to enumerate namespaces depending on how they were created.

## Prerequisites

- Linux with iproute2 installed
- Root access for full visibility

## List Named Namespaces with ip netns

Named namespaces (created with `ip netns add`) are stored as bind-mount files in `/var/run/netns/`:

```bash
# List all named network namespaces

ip netns list

# Example output:
# ns2 (id: 1)
# ns1 (id: 0)
```

The ID shown is the kernel namespace identifier.

## List Namespace Files Directly

```bash
# List namespace files in the filesystem
ls -la /var/run/netns/

# Check with file type
ls -lh /var/run/netns/
```

## Get Detailed Namespace Info

```bash
# Show namespace with its ID and interface count
ip netns list -v 2>/dev/null || ip netns list
```

## List ALL Namespaces (Including Container Namespaces)

Containers (Docker, Kubernetes) create unnamed namespaces that do NOT appear in `ip netns list`. Use `lsns` to see all:

```bash
# List all network namespaces on the system (requires util-linux)
lsns -t net

# Example output:
#         NS TYPE  NPROCS   PID USER    NETNSID NSFS COMMAND
# 4026531992 net      123     1 root unassigned      /sbin/init
# 4026532189 net        1  1234 root          0      /pause
# 4026532250 net        2  5678 root          1      nginx
```

## Inspect a Running Process Namespace

```bash
# Find the network namespace of a specific process (PID 1234)
ls -la /proc/1234/ns/net

# The symlink target is the namespace inode (e.g., net:[4026532189])
```

## Check Which Namespace You Are In

```bash
# Show the current process's network namespace inode
readlink /proc/self/ns/net

# Compare with another namespace file
readlink /var/run/netns/ns1
```

## List Interfaces Inside Each Namespace

```bash
# Loop through all named namespaces and list their interfaces
for ns in $(ip netns list | awk '{print $1}'); do
    echo "=== Namespace: $ns ==="
    ip netns exec "$ns" ip link show
done
```

## Associate Processes with Namespaces

```bash
# Find all processes in a specific namespace (by inode)
NS_INODE=$(readlink /var/run/netns/ns1 | tr -d 'net:[]')
lsns -t net | grep "$NS_INODE"
```

## Docker and Kubernetes Namespace Visibility

Docker creates network namespaces but does not mount them in `/var/run/netns` by default. To make a Docker container's namespace visible to `ip netns`:

```bash
# Get the container's PID
CONTAINER_PID=$(docker inspect --format '{{.State.Pid}}' my_container)

# Create a bind-mount link in /var/run/netns
mkdir -p /var/run/netns
ln -sf /proc/$CONTAINER_PID/ns/net /var/run/netns/my_container

# Now the container appears in ip netns list
ip netns list
ip netns exec my_container ip addr
```

## Conclusion

`ip netns list` shows named namespaces created with iproute2, while `lsns -t net` reveals all network namespaces including those created by containers. For deep inspection, check `/proc/<pid>/ns/net` to find the namespace a specific process is running in. Docker containers require a manual bind-mount step to appear in `ip netns`.
