# How to Fix 'clock skew detected' in Ceph Monitors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitors, NTP, Troubleshooting

Description: Fix clock skew warnings in Ceph monitors by synchronizing time with NTP and configuring the mon_clock_drift_allowed tolerance threshold.

---

## Why Clock Skew Is a Problem for Ceph

Ceph monitors use distributed consensus (Paxos) to maintain cluster state. This algorithm requires that all monitor nodes have reasonably synchronized clocks. When clocks drift too far apart, Ceph reports a warning and, if the skew is large enough, monitors can lose quorum entirely, causing the cluster to become unavailable.

The default tolerance is 0.05 seconds (50ms). Any monitor exceeding this drift triggers the warning.

## Identifying the Problem

Check cluster health:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

You may see:

```text
HEALTH_WARN clock skew detected on mon.b, mon.c
clock skew 0.123045 > max 0.05 on mon.b
```

Or check monitor status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mon stat
```

## Step 1 - Check Current System Time on Each Node

SSH into each node hosting a MON pod and check the time:

```bash
date
timedatectl status
```

Example output showing a problem:

```text
               Local time: Tue 2026-03-31 10:05:33 UTC
           Universal time: Tue 2026-03-31 10:05:33 UTC
                 RTC time: Tue 2026-03-31 10:05:45 UTC
                Time zone: UTC (UTC, +0000)
System clock synchronized: no
              NTP service: inactive
```

`NTP service: inactive` indicates the problem.

## Step 2 - Install and Enable NTP (systemd-timesyncd)

On Debian/Ubuntu-based nodes:

```bash
sudo apt-get install -y systemd-timesyncd
sudo systemctl enable systemd-timesyncd
sudo systemctl start systemd-timesyncd
timedatectl set-ntp true
```

On RHEL/CentOS/Rocky Linux nodes:

```bash
sudo dnf install -y chrony
sudo systemctl enable chronyd
sudo systemctl start chronyd
```

## Step 3 - Verify NTP Synchronization

Check synchronization status:

```bash
timedatectl show --property=NTPSynchronized --value
```

Expected output: `yes`

For chrony:

```bash
chronyc tracking
```

Look for a low `System time` offset value (milliseconds or less).

## Step 4 - Force an Immediate Time Sync

If the clock is far off and you need immediate synchronization:

```bash
# For systemd-timesyncd
sudo systemctl stop systemd-timesyncd
sudo ntpdate -u pool.ntp.org
sudo systemctl start systemd-timesyncd
```

Or with chrony:

```bash
sudo chronyc makestep
```

## Step 5 - Configure Custom NTP Servers

If your nodes are in a private network without internet access, configure an internal NTP server.

For systemd-timesyncd, edit `/etc/systemd/timesyncd.conf`:

```text
[Time]
NTP=192.168.1.10 192.168.1.11
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org
```

Restart the service:

```bash
sudo systemctl restart systemd-timesyncd
```

For chrony, edit `/etc/chrony.conf`:

```text
server 192.168.1.10 iburst
server 192.168.1.11 iburst
```

Then restart:

```bash
sudo systemctl restart chronyd
```

## Step 6 - Adjust Ceph's Clock Drift Tolerance

If you cannot achieve tight NTP synchronization (e.g., virtualized environments with inherent clock jitter), increase Ceph's tolerance:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph tell mon.* injectargs '--mon-clock-drift-allowed=0.2'
```

This sets the tolerance to 200ms. For a permanent change, update the Rook `CephCluster` spec:

```yaml
spec:
  cephConfig:
    mon:
      mon_clock_drift_allowed: "0.2"
```

Apply the change:

```bash
kubectl apply -f cephcluster.yaml
```

## Step 7 - Special Considerations for Kubernetes Environments

In Kubernetes, MON pods run on nodes where time is controlled by the host OS. Ensure:

1. **All worker nodes** have NTP enabled, not just the master.
2. If using cloud VMs, ensure the hypervisor time synchronization is enabled.
3. For VMware: enable VMware Tools time synchronization.
4. For AWS: ensure the EC2 instance uses the AWS Time Sync Service (`169.254.169.123`).

Check the MON pod's time from within the pod:

```bash
kubectl -n rook-ceph exec -it <mon-pod-name> -- date
```

Compare this to the host node's time to verify there's no container-level drift.

## Step 8 - Verify the Warning Is Gone

After fixing NTP:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

The clock skew warning should disappear within a few minutes of the clocks converging.

## Summary

Clock skew in Ceph monitors is caused by unsynchronized system clocks on nodes hosting MON pods. The fix is to enable and configure NTP (chrony or systemd-timesyncd) on all nodes, force an immediate time correction, and verify synchronization. In environments with unavoidable clock jitter, raise `mon_clock_drift_allowed` to prevent false warnings while still protecting against real drift issues.
