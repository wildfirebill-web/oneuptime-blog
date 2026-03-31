# How to Recover Ceph After OS Reinstall on Monitor Nodes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Disaster Recovery, Monitor, OS Reinstall, Storage

Description: Learn how to recover a Ceph cluster after operating system reinstallation on monitor nodes, restore monitor identities, and re-establish quorum.

---

## The Challenge of OS Reinstall on Monitor Nodes

When an OS is reinstalled on a Ceph monitor node, the monitor data directory (typically `/var/lib/ceph/mon/`) is lost. Without this directory, the monitor cannot rejoin the cluster because it has lost its identity, authentication keys, and monitor map.

Recovery requires either:
1. Restoring the monitor data directory from backup
2. Using a surviving monitor's data to rebuild the affected monitor

## Step 1: Assess What Survived

Check how many monitors still have their data intact:

```bash
ceph mon stat
ceph quorum_status
```

If at least one monitor has quorum, you can rebuild the others. If all monitors lost their data, see the monitor database rebuild guide.

## Step 2: Get the Current Monitor Map

From a surviving monitor:

```bash
ceph mon getmap -o /tmp/monmap.bin
monmaptool --print /tmp/monmap.bin
```

Note the addresses and names of all monitors.

## Step 3: Retrieve Authentication Keys

On a surviving monitor, extract the bootstrap key for the new monitor:

```bash
ceph auth get mon. -o /tmp/mon.keyring
cat /tmp/mon.keyring
```

## Step 4: Reinitialize the Monitor

On the reinstalled node, recreate the monitor data directory:

```bash
# Create the monitor directory
mkdir -p /var/lib/ceph/mon/ceph-$(hostname)

# Initialize using the monitor map and keyring from surviving monitors
ceph-mon --mkfs \
  -i $(hostname) \
  --monmap /tmp/monmap.bin \
  --keyring /tmp/mon.keyring
```

## Step 5: Set Correct Ownership and Start

```bash
chown -R ceph:ceph /var/lib/ceph/mon/
systemctl start ceph-mon@$(hostname)
systemctl enable ceph-mon@$(hostname)
```

## Step 6: Verify Monitor Joined Quorum

```bash
ceph mon stat
ceph quorum_status | python3 -m json.tool | grep quorum_names
```

All three monitor names should appear in the quorum.

## Rook Monitor Recovery After Node OS Reinstall

In Rook, monitors are managed as Kubernetes pods. After an OS reinstall on the node:

1. Ensure the node rejoins the Kubernetes cluster
2. Check that the node's labels are restored (especially `topology` labels)
3. Let the Rook operator reconcile

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon
kubectl -n rook-ceph describe cephcluster rook-ceph
```

If a monitor pod is stuck or the operator is not reprovisioning:

```bash
kubectl -n rook-ceph delete pod -l app=rook-ceph-mon,ceph_daemon_id=a
```

The operator will reprovision the monitor using data from the remaining monitors.

## Preventing Monitor Data Loss

Back up monitor data regularly:

```bash
#!/bin/bash
# Backup monitor data
tar -czf /backup/mon-$(hostname)-$(date +%Y%m%d).tar.gz \
  /var/lib/ceph/mon/ceph-$(hostname)/
```

## Summary

Recovering Ceph monitors after OS reinstall involves reinitializing the monitor directory using the monitor map and keyring from surviving monitors. As long as quorum survives, recovery is straightforward. In Rook environments, the operator automates monitor recovery when nodes rejoin the Kubernetes cluster. Regular backups of the monitor data directory provide an additional safety net for complete cluster recovery.
