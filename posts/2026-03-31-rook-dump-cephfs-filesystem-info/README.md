# How to Dump CephFS Filesystem Info

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS

Description: Learn how to use ceph fs dump and related commands to inspect the full CephFS filesystem map, MDS state, and configuration in Rook deployments.

---

## Overview of ceph fs dump

`ceph fs dump` prints the complete filesystem map (FSMap) which is the authoritative record of all CephFS filesystems, their pools, MDS assignments, compatibility flags, and configuration. This is the primary diagnostic command when investigating CephFS issues at a global level.

Run from the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs dump
```

## Sample Dump Output

```text
epoch 42
flags
compat compat={},rocompat={},incompat={1=base v0.20,2=client writeable ranges,3=default file layouts on dirs,4=dir inode in separate object,5=mds uses versioned encoding,6=dirfrag is stored in omap,8=no anchor table,9=file layout v2,10=snaprealm v2}

Filesystem 'myfs' (1)
fs_name    myfs
epoch      36
flags      12
created    2026-01-15T12:00:00.000000+0000
modified   2026-03-01T08:00:00.000000+0000
tableserver   0
root    0
session_timeout   60
session_autoclose 300
max_file_size     1099511627776
last_failure   0
last_failure_osd_epoch  0
compat    compat={},rocompat={},incompat={...}
metadata_pool     1
inline_data   disabled
data_pools    [2 ]
...
```

## Key Fields Explained

| Field | Meaning |
|---|---|
| `epoch` | FSMap version - increments on every change |
| `flags` | Filesystem state flags |
| `session_timeout` | Client session timeout in seconds |
| `session_autoclose` | Idle session autoclose time |
| `max_file_size` | Maximum allowed file size (bytes) |
| `metadata_pool` | Pool ID for metadata |
| `data_pools` | List of data pool IDs |
| `inline_data` | Whether inline data is enabled |

## Dumping in JSON Format

For scripting and parsing:

```bash
ceph fs dump --format json-pretty
```

Sample JSON excerpt:

```json
{
  "epoch": 42,
  "default_fscid": 1,
  "filesystems": [
    {
      "mdsmap": {
        "fs_name": "myfs",
        "enabled": true,
        "epoch": 36,
        "metadata_pool": 1,
        "data_pools": [2],
        "max_mds": 1,
        "session_timeout": 60,
        "max_file_size": 1099511627776,
        "in": [0],
        "up": {"mds_0": 4107}
      },
      "id": 1
    }
  ],
  "standbys": [
    {"name": "myfs-b", "rank": -1, "state": "up:standby"}
  ]
}
```

## Checking MDS Assignments

The dump output shows which MDS daemon is serving each rank:

```bash
ceph fs dump --format json | jq '.filesystems[0].mdsmap.up'
```

Sample output:

```json
{"mds_0": 4107}
```

The value `4107` is the daemon GID. Cross-reference with MDS status:

```bash
ceph mds stat
```

## Checking the Epoch

The FSMap epoch increments every time a change is made. A rapidly incrementing epoch during normal operation may indicate MDS instability:

```bash
# Watch the epoch
watch -n 2 'ceph fs dump | grep "^epoch"'
```

## Getting Filesystem Map for a Specific Filesystem

To see only the map for a specific filesystem:

```bash
ceph fs get myfs
```

This is a shorter output focused on `myfs` settings without the global FSMap header.

## Checking Compatibility

CephFS compatibility flags indicate the minimum Ceph version required for clients to mount the filesystem:

```bash
ceph fs dump | grep -A2 "compat"
```

If you see many `incompat` entries, older Ceph client versions may not be able to mount the filesystem.

## Summary

`ceph fs dump` provides the complete FSMap including all filesystems, their pool assignments, MDS state, configuration settings, and compatibility flags. Use `--format json` for programmatic parsing and `jq` for extracting specific fields. Monitor the FSMap epoch to detect unexpected churn. Use `ceph fs get <name>` for a focused view of a single filesystem's settings.
