# How to Troubleshoot cephadm Deployment Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, cephadm, Troubleshooting, Deployment, Debugging

Description: Diagnose and fix common cephadm deployment failures including daemon start failures, SSH errors, container pull issues, and configuration problems.

---

## First Stop: `ceph orch ps`

When a deployment fails, start by checking daemon status:

```bash
ceph orch ps
```

Look for daemons with `error` or `stopped` status. Get details on a specific daemon:

```bash
ceph orch ps --daemon-type osd --format yaml
```

## Checking Daemon Logs

```bash
# View logs for a specific daemon
ceph log last ceph-osd.3 100

# Or use journalctl on the host
journalctl -u ceph-osd@3.service --no-pager | tail -50

# View via cephadm
cephadm logs --name osd.3
```

## Common Error: Daemon Won't Start

**Symptom:** `ceph orch ps` shows `error` for a daemon.

```bash
# Check the container exit code
docker ps -a | grep ceph-osd
podman ps -a | grep ceph-osd

# View container logs
docker logs <container-id>
# or
podman logs <container-id>
```

## Common Error: SSH Connectivity Failure

```bash
ceph orch host ls
# Shows "offline" or connectivity error

# Test directly
ceph cephadm check-host --addr 10.0.0.25
```

Fix: re-distribute the SSH key and verify `authorized_keys` permissions.

## Common Error: Image Pull Failure

```bash
# Check if the image exists
cephadm pull --image quay.io/ceph/ceph:v18.2.0

# Error: unauthorized
# Fix: configure registry credentials
ceph cephadm registry-login \
  --registry-url https://registry.example.com \
  --registry-username user \
  --registry-password pass
```

## Common Error: OSD Deployment Fails

**Symptom:** OSDs in `unmanaged` state or not created.

```bash
# Check available devices
ceph orch device ls

# Devices may be listed as "rejected" - check reason
ceph orch device ls --wide

# Common reasons for rejection:
# - Partition table exists
# - LVM data present
# - Device too small (< 5GB)
# - Device already in use

# Zap a device manually
ceph orch device zap node1 /dev/sdb --force
```

## Common Error: Monitor Can't Form Quorum

```bash
ceph mon stat
ceph mon dump | grep quorum
```

```bash
# Check mon daemon logs
cephadm logs --name mon.node1

# Verify network connectivity between monitors
ping node2 && ping node3
```

## Using cephadm in Debug Mode

For verbose cephadm output:

```bash
cephadm --verbose <command>
```

For MGR orchestrator debug:

```bash
ceph config set mgr mgr/cephadm/log_to_cluster_level debug
ceph log last ceph-mgr 200 | grep -i error
```

Restore normal logging:

```bash
ceph config set mgr mgr/cephadm/log_to_cluster_level info
```

## Checking the MGR Orchestrator Status

```bash
ceph orch status
```

If orchestrator is not available:

```bash
ceph mgr module enable cephadm
ceph orch set backend cephadm
```

## Summary

Start cephadm deployment troubleshooting with `ceph orch ps` to find errored daemons, then use `cephadm logs` or `journalctl` for daemon-level details. For SSH issues, run `ceph cephadm check-host`; for image issues, test with `cephadm pull`. OSD failures are often caused by non-empty devices - use `ceph orch device ls --wide` to see rejection reasons and `device zap` to clear them.
