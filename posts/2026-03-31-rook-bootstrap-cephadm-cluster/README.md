# How to Bootstrap a Ceph Cluster with cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, cephadm, Bootstrap, Storage, Deployment

Description: Step-by-step guide to bootstrapping a new Ceph cluster with cephadm, covering prerequisites, the bootstrap command, and initial configuration.

---

## Bootstrap Overview

The `cephadm bootstrap` command creates the foundation of a new Ceph cluster on a single host. It deploys the first monitor and manager, generates the cluster FSID and keyrings, installs the `ceph` CLI inside the container, and optionally deploys the Ceph Dashboard. All subsequent cluster expansion (adding hosts, deploying OSDs, services) builds on this bootstrapped foundation.

## Pre-Bootstrap Checklist

Before running bootstrap, verify each host:

```bash
# Check Python 3 is available
python3 --version

# Check Podman or Docker
podman --version || docker --version

# Check NTP sync
timedatectl status | grep "synchronized: yes"

# Check hostname resolves
hostname -f
ping $(hostname -f)

# Verify no existing Ceph processes
systemctl list-units --state=running | grep ceph
```

## Installing cephadm

```bash
# Fetch cephadm bootstrap script
curl --silent --remote-name --location \
  https://github.com/ceph/ceph/raw/main/src/cephadm/cephadm

chmod +x cephadm

# Use cephadm to add the Ceph package repos
sudo ./cephadm add-repo --release quincy

# Install the cephadm package (makes it available in PATH)
sudo ./cephadm install
```

## The Bootstrap Command

```bash
sudo cephadm bootstrap \
  --mon-ip 192.168.1.10 \
  --cluster-network 192.168.2.0/24 \
  --initial-dashboard-user admin \
  --initial-dashboard-password SecurePass123! \
  --dashboard-password-noupdate \
  --allow-fqdn-hostname \
  --log-to-file
```

Key flags:
- `--mon-ip`: IP address the monitor will bind to
- `--cluster-network`: Dedicated backend network for OSD replication traffic
- `--allow-fqdn-hostname`: Required if your hostname is a FQDN
- `--log-to-file`: Write logs to `/var/log/ceph/` in addition to journald

## Bootstrap Output

On success, cephadm prints:

```text
INFO:cephadm:Ceph Dashboard is now available at:
             URL: https://192.168.1.10:8443/
            User: admin
        Password: SecurePass123!

INFO:cephadm:You can access the Ceph CLI with:
    sudo /usr/sbin/cephadm shell --fsid a1b2c3d4-...
```

## Post-Bootstrap Verification

```bash
# Enter the ceph shell
sudo cephadm shell

# Check cluster status
ceph status

# Verify monitor is in quorum
ceph mon stat

# Check manager is active
ceph mgr stat
```

Expected status:

```text
cluster:
  id:     a1b2c3d4-e5f6-7890-abcd-ef1234567890
  health: HEALTH_WARN
          OSD count 0 < osd_pool_default_min_size 1

services:
  mon: 1 daemons, quorum node1 (age 2m)
  mgr: node1.abcdef(active, since 1m)
  osd: 0 osds: 0 up, 0 in
```

The `HEALTH_WARN` about OSD count is expected before adding storage.

## Configuring the ceph CLI on the Host

Install the `ceph` CLI directly on the host without entering the container:

```bash
sudo cephadm install ceph-common

# Test direct CLI access
ceph status
```

## Enabling Useful Modules

```bash
# Enable the Prometheus metrics module
ceph mgr module enable prometheus

# Enable the telemetry module (optional, sends anonymized stats to Ceph team)
ceph telemetry on

# Enable PG autoscaler
ceph mgr module enable pg_autoscaler
ceph config set global osd_pool_default_pg_autoscale_mode on
```

## Summary

The `cephadm bootstrap` command bootstraps a complete Ceph cluster foundation from a single command on the first host. It handles FSID generation, keyring creation, monitor and manager deployment, and dashboard setup. After bootstrap, verifying quorum status and monitor health confirms the foundation is ready for the next steps of adding hosts and deploying OSDs. A successful bootstrap is the prerequisite for all subsequent cephadm-managed cluster operations.
