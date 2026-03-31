# How to Use cephadm Shell for Cluster Administration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, cephadm, Shell, Admin, CLI, Container

Description: Use the cephadm shell command to run Ceph CLI tools inside a container without installing ceph-common packages directly on the host.

---

## What Is cephadm Shell?

`cephadm shell` launches a containerized shell with all Ceph CLI tools pre-installed. This is useful when you have not installed `ceph-common` on your admin node, or when you need a specific Ceph version's tools to match the cluster.

## Basic Usage

Launch an interactive Ceph admin shell:

```bash
sudo cephadm shell
```

This drops you into a container with `ceph`, `rbd`, `radosgw-admin`, and all other Ceph tools available.

Inside the shell:

```bash
ceph status
ceph df
rbd ls mypool
```

Exit with `exit` or Ctrl+D.

## Running a Single Command

Execute a command without entering an interactive shell:

```bash
sudo cephadm shell -- ceph status
sudo cephadm shell -- ceph health detail
sudo cephadm shell -- radosgw-admin user list
```

The `--` separator is required to pass arguments to the command inside the container.

## Passing Environment Variables

```bash
sudo cephadm shell --env CEPH_ARGS="--cluster mycluster" -- ceph status
```

## Mounting Additional Paths

Mount extra directories into the shell container:

```bash
sudo cephadm shell --mount /tmp/scripts:/scripts -- bash /scripts/my-admin-script.sh
```

## Using a Specific Image Version

Run the shell with a different Ceph version:

```bash
sudo cephadm shell --image quay.io/ceph/ceph:v17.2.7 -- ceph version
```

This is useful for testing commands against a different version or troubleshooting compatibility.

## Using cephadm Shell in Scripts

For automation without interactive sessions:

```bash
#!/bin/bash
CLUSTER_STATUS=$(sudo cephadm shell -- ceph status --format json)
echo "$CLUSTER_STATUS" | python3 -c "
import json, sys
d = json.load(sys.stdin)
print('Health:', d['health']['status'])
"
```

## Comparing to Direct ceph CLI

If `ceph-common` is installed:

```bash
# Direct (requires ceph-common)
ceph status

# Via cephadm shell (no ceph-common needed)
cephadm shell -- ceph status
```

The containerized version always uses the exact version matching the cluster's containers.

## File Access Inside the Shell

The shell container automatically mounts:
- `/etc/ceph/` - config and keyrings
- `/var/log/ceph/` - cluster logs

So you can immediately inspect logs:

```bash
sudo cephadm shell -- tail -100 /var/log/ceph/ceph.log
```

## Summary

`cephadm shell` provides a containerized Ceph admin environment without requiring `ceph-common` on the host. Use `cephadm shell -- <command>` for one-off admin tasks, `--mount` to bring in local scripts, and `--image` to pin a specific Ceph version. It is ideal for scripted administration and environments where package installation on the host is restricted.
