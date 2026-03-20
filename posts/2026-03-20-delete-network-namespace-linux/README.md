# How to Delete a Network Namespace on Linux

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Namespaces, iproute2, Networking, Cleanup, System Administration

Description: Delete network namespaces on Linux using ip netns delete, clean up associated interfaces, and understand what happens to processes running inside deleted namespaces.

## Introduction

When you no longer need a network namespace, you should delete it to free resources and avoid confusion. Deleting a namespace removes the bind-mount file, but processes running inside it continue until they exit. This guide covers safe deletion and cleanup.

## Prerequisites

- Linux with iproute2
- Root access
- Understanding of what is running inside the namespace before deletion

## Delete a Named Namespace

```bash
# Delete a specific namespace by name

ip netns delete ns1

# Verify it is gone
ip netns list
```

## What Happens on Deletion

When you delete a namespace:
1. The file `/var/run/netns/<name>` is removed (the bind-mount is released)
2. The namespace persists as long as any process or file descriptor references it
3. Processes already running inside the namespace continue to run but are orphaned
4. Virtual interfaces (veth) created for the namespace may remain on the host side

## Clean Up veth Interfaces Before Deleting

If the namespace has a veth pair, the host-side interface remains after namespace deletion. Clean it up:

```bash
# Before deleting the namespace, identify connected veth interfaces
ip link show

# Example: if veth0 was connected to a namespace, delete it
ip link delete veth0
# Note: deleting one side of a veth pair automatically deletes the other

# Then delete the namespace
ip netns delete ns1
```

## Delete All Named Namespaces

```bash
# Loop through all namespaces and delete them
for ns in $(ip netns list | awk '{print $1}'); do
    echo "Deleting namespace: $ns"
    ip netns delete "$ns"
done
```

## Check for Processes in a Namespace Before Deleting

```bash
# List processes running in a namespace
ip netns pids ns1

# If there are PIDs, consider what they are before deleting
ip netns pids ns1 | xargs -I{} cat /proc/{}/cmdline | tr '\0' ' '
```

## Delete the Namespace File Manually

If `ip netns delete` fails (e.g., the namespace is still referenced), you can unmount manually:

```bash
# Manually unmount the namespace file
umount /var/run/netns/ns1

# Then remove the file
rm /var/run/netns/ns1
```

## Cleanup After Container Tools

Docker and Kubernetes do not store namespaces in `/var/run/netns`, so `ip netns delete` does not apply. However, if you created manual symlinks:

```bash
# Remove a manually created symlink for a Docker container namespace
rm /var/run/netns/my_container
```

## Verify Complete Cleanup

```bash
# Confirm no named namespaces remain
ip netns list

# Confirm no orphaned veth interfaces remain
ip link show type veth

# Check /var/run/netns directory is empty
ls /var/run/netns/
```

## Conclusion

Deleting a network namespace with `ip netns delete` removes the namespace name and mount point. Always clean up associated veth interfaces on the host side first, and check for running processes with `ip netns pids` before deletion. The kernel retains the namespace until all processes and file descriptors referencing it are released.
