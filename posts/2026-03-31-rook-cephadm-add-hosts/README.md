# How to Add Hosts to a Ceph Cluster with cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, cephadm, Storage, Cluster, Scaling

Description: Learn how to add new hosts to an existing cephadm-managed Ceph cluster, including SSH key distribution, host labeling, and service placement.

---

## Adding Hosts in cephadm

After bootstrapping the first monitor node, cephadm makes it easy to expand the cluster by adding more hosts. cephadm uses SSH to communicate with hosts and deploy Ceph daemon containers. Each new host requires the cluster's SSH public key before it can be added.

## Distributing the SSH Key

cephadm generates an SSH key pair during bootstrap. Copy the public key to each new host:

```bash
# View the cephadm SSH public key
ceph cephadm get-pub-key

# Copy to a new host (root access required)
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.11
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.12
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.13
```

Alternatively, use your existing SSH infrastructure:

```bash
# If ceph.pub is on the host, manually append to authorized_keys
cat /etc/ceph/ceph.pub | ssh root@192.168.1.11 \
  "cat >> /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys"
```

## Adding Hosts to the Cluster

```bash
# Add hosts (IP address is optional if DNS resolves hostname)
ceph orch host add node2 192.168.1.11
ceph orch host add node3 192.168.1.12
ceph orch host add node4 192.168.1.13

# Verify all hosts are managed
ceph orch host ls
```

Output:

```text
HOST   ADDR          LABELS  STATUS
node1  192.168.1.10  _admin
node2  192.168.1.11
node3  192.168.1.12
node4  192.168.1.13
```

## Labeling Hosts

Labels organize hosts for service placement. Common labels include `mon`, `osd`, `mgr`, `rgw`, and `_admin`.

```bash
# Mark a host as admin (receives /etc/ceph config files)
ceph orch host label add node2 _admin

# Label hosts for specific services
ceph orch host label add node1 mon
ceph orch host label add node2 mon
ceph orch host label add node3 mon

ceph orch host label add node2 osd
ceph orch host label add node3 osd
ceph orch host label add node4 osd

# View labels
ceph orch host ls --detail
```

## Deploying Monitors on New Hosts

After adding hosts with the `mon` label, deploy additional monitors:

```bash
# Apply 3-monitor placement (uses hosts with mon label)
ceph orch apply mon "node1,node2,node3"

# Or use label-based placement
ceph orch apply mon --placement="label:mon"

# Verify 3 monitors are in quorum
ceph mon stat
```

## Checking Host Health

```bash
# Check for any issues with added hosts
ceph orch host check-hosts

# View per-host daemon inventory
ceph orch ps --host node2

# Verify SSH connectivity
ceph cephadm check-host node2
```

## Removing a Host

Before removing a host, drain its daemons:

```bash
# Drain all daemons from a host
ceph orch host drain node4

# Wait for drain to complete
ceph orch ps --host node4

# Remove the host once no daemons remain
ceph orch host rm node4
```

## Updating Host Addresses

If a host IP changes:

```bash
# Update the host address
ceph orch host update node2 addr=192.168.1.20

# Re-copy SSH key to new address if needed
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.20
```

## Summary

Adding hosts to a cephadm-managed Ceph cluster requires distributing the cluster's SSH public key, registering the host with `ceph orch host add`, and optionally assigning labels that control service placement. Labels are the primary mechanism for targeting specific hosts for monitors, OSDs, RGW gateways, and other Ceph services. Keeping host labels accurate and running `ceph orch host check-hosts` regularly ensures cephadm can reliably manage all cluster nodes.
