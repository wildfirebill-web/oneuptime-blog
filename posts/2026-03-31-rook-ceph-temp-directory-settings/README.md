# How to Configure Temporary Directory Settings in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Configuration, TempDirectory, Storage, Tuning

Description: Learn how to configure Ceph temporary directory and run directory settings for daemons, admin sockets, and PID files to match your system layout.

---

## Temporary and Run Directories in Ceph

Ceph daemons use several directories for runtime artifacts:
- **Admin socket** (`asok`): A Unix domain socket used for live daemon introspection and command injection
- **PID file**: Records the daemon's process ID for init system management
- **Run directory**: Parent directory for socket files and PIDs
- **Temp directory**: Used by some Ceph operations for temporary data storage

Configuring these correctly is important for systems with non-standard layouts, security-hardened environments, or containerized deployments where default paths may not be appropriate.

## Default Directory Locations

Ceph defaults to:
- Run directory: `/var/run/ceph/`
- Admin sockets: `/var/run/ceph/$cluster-$type.$id.asok`
- PID files: `/var/run/ceph/$cluster-$type.$id.pid`
- Temp dir: `/tmp/`

## Configuring the Run Directory

```bash
# Set a custom run directory
ceph config set global run_dir /run/ceph

# Or in ceph.conf
cat >> /etc/ceph/ceph.conf << 'EOF'
[global]
run_dir = /run/ceph
EOF
```

Create the directory with correct ownership:

```bash
sudo mkdir -p /run/ceph
sudo chown ceph:ceph /run/ceph
sudo chmod 750 /run/ceph
```

## Admin Socket Configuration

The admin socket enables live interaction with running daemons:

```ini
[global]
; Admin socket path template
admin_socket = /var/run/ceph/$cluster-$type.$id.$pid.$cctid.asok
```

The variables expand as:
- `$cluster` - cluster name (default: `ceph`)
- `$type` - daemon type (`osd`, `mon`, `mds`)
- `$id` - daemon ID (`0`, `mon-a`, etc.)
- `$pid` - process ID

```bash
# List admin sockets for running daemons
ls /var/run/ceph/*.asok

# Use an admin socket
ceph daemon /var/run/ceph/ceph-osd.0.asok config show

# In Rook - access via toolbox
kubectl exec -it rook-ceph-tools -n rook-ceph -- \
  ceph daemon osd.0 perf dump
```

## Configuring the Temp Directory

Some Ceph operations write temporary files (e.g., CRUSH map manipulation, export operations):

```bash
# Set a custom temp directory
ceph config set global tmp_dir /var/lib/ceph/tmp

# Create and secure the directory
sudo mkdir -p /var/lib/ceph/tmp
sudo chown ceph:ceph /var/lib/ceph/tmp
sudo chmod 750 /var/lib/ceph/tmp
```

For containerized deployments where `/tmp` may be RAM-backed (tmpfs), using a persistent path prevents failures when exporting large CRUSH maps or OSD metadata.

## Per-Daemon Run Directories

In environments running multiple Ceph clusters (multi-cluster hosts), use per-cluster run directories:

```ini
[global]
run_dir = /var/run/ceph/$cluster
```

```bash
sudo mkdir -p /var/run/ceph/ceph
sudo chown ceph:ceph /var/run/ceph/ceph
```

## systemd tmpfiles Configuration

On systemd-based systems, ensure run directories persist across reboots via tmpfiles.d:

```bash
cat > /etc/tmpfiles.d/ceph.conf << 'EOF'
d /var/run/ceph 0755 ceph ceph -
d /var/run/ceph/ceph 0755 ceph ceph -
d /var/lib/ceph/tmp 0750 ceph ceph -
EOF

# Apply immediately
systemd-tmpfiles --create /etc/tmpfiles.d/ceph.conf
```

## Rook Considerations

In Rook, admin sockets are inside daemon containers. Access them via the toolbox:

```bash
# The toolbox has access to daemon sockets
kubectl exec -it rook-ceph-tools -n rook-ceph -- bash

# List available sockets
ls /run/ceph/

# Interact with a daemon
ceph daemon osd.0 config show | grep tmp_dir
```

## Summary

Ceph's temporary and run directory settings control where daemon admin sockets, PID files, and temporary operation files are stored. Configuring these paths is important for security-hardened systems with restricted `/tmp` access, multi-cluster deployments needing namespace separation, and containerized environments where default paths may conflict. Ensuring directories exist with correct `ceph:ceph` ownership and that systemd tmpfiles.d recreates them on reboot prevents daemon startup failures and simplifies live diagnostics through admin socket commands.
