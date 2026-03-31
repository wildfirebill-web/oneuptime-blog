# How to Add Hosts with cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, cephadm, Host, Cluster, Expansion

Description: Add new nodes to an existing Ceph cluster managed by cephadm, including SSH key distribution, host registration, and label assignment.

---

## Why Add Hosts?

Adding hosts to a cephadm-managed Ceph cluster is how you expand capacity (more OSDs), improve availability (more monitors), and scale services (more RGW or MDS daemons). cephadm handles the container deployment on new hosts automatically once they are registered.

## Prerequisites

- A running cephadm-managed cluster
- SSH access from the admin node to the new host (root or sudo)
- The new host meets Ceph hardware requirements

## Step 1: Distribute the Ceph SSH Key

cephadm uses a dedicated SSH key to connect to managed hosts. Copy it to the new host:

```bash
# View the public key
cat /etc/ceph/ceph.pub

# Copy to the new host
ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.0.0.20
```

If you cannot use ssh-copy-id:

```bash
ssh root@10.0.0.20 \
  "mkdir -p ~/.ssh && echo '$(cat /etc/ceph/ceph.pub)' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

## Step 2: Verify SSH Connectivity

Test that cephadm can connect:

```bash
ceph cephadm check-host --addr 10.0.0.20
```

This checks SSH connectivity, Python version, container runtime, and hostname resolution.

## Step 3: Add the Host to the Cluster

```bash
ceph orch host add node4 10.0.0.20
```

If the hostname differs from what DNS resolves:

```bash
ceph orch host add node4 10.0.0.20 --addr 10.0.0.20
```

Verify the host was added:

```bash
ceph orch host ls
```

## Step 4: Assign Labels

Labels control which services cephadm deploys on a host:

```bash
# Add labels
ceph orch host label add node4 osd
ceph orch host label add node4 mon
```

Common labels:

| Label | Purpose |
|-------|---------|
| `_admin` | Admin host (receives ceph.conf and keyrings) |
| `mon` | Monitor placement |
| `osd` | OSD node |
| `rgw` | RGW node |
| `mds` | MDS node |

## Step 5: Deploy Services on the New Host

After adding, cephadm may auto-deploy services based on placement rules. To manually add an OSD:

```bash
ceph orch daemon add osd node4:/dev/sdb
```

Add a monitor:

```bash
ceph orch apply mon "node1,node2,node3,node4"
```

## Step 6: Verify

```bash
ceph orch ps --hostname node4
ceph status
```

## Summary

Adding a host with cephadm requires distributing the Ceph SSH public key to the new node, then running `ceph orch host add`. Assign labels to control service placement and use `ceph orch daemon add` for immediate service deployment. Verify with `ceph orch host ls` and `ceph orch ps --hostname`.
