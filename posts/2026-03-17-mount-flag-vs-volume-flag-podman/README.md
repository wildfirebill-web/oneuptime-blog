# How to Use the --mount Flag vs --volume Flag in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Volumes, Mount, CLI

Description: Learn the differences between the --mount and --volume flags in Podman and when to use each one.

---

> Podman provides two syntaxes for mounting volumes: the concise --volume (-v) flag and the explicit --mount flag. Understanding their differences helps you choose the right approach.

Both `--volume` and `--mount` achieve the same result of attaching storage to containers, but they differ in syntax, readability, and behavior for edge cases. This guide compares both approaches with practical examples.

---

## The --volume (-v) Flag

The `-v` flag uses a colon-separated string with three fields:

```bash
# Syntax: -v source:destination:options

# Named volume
podman run -d -v mydata:/app/data docker.io/library/nginx:latest

# Bind mount
podman run -d -v /home/user/config:/etc/nginx/conf.d:ro docker.io/library/nginx:latest

# With multiple options
podman run -d -v /home/user/data:/data:Z,rw docker.io/library/nginx:latest
```

## The --mount Flag

The `--mount` flag uses key=value pairs separated by commas:

```bash
# Named volume
podman run -d \
  --mount type=volume,source=mydata,target=/app/data \
  docker.io/library/nginx:latest

# Bind mount
podman run -d \
  --mount type=bind,source=/home/user/config,target=/etc/nginx/conf.d,readonly \
  docker.io/library/nginx:latest

# tmpfs mount
podman run -d \
  --mount type=tmpfs,target=/tmp,tmpfs-size=100m \
  docker.io/library/nginx:latest
```

## Key Differences

| Feature | --volume (-v) | --mount |
|---------|--------------|---------|
| Syntax | Colon-separated | Key-value pairs |
| Readability | Concise | Explicit |
| Auto-create dirs | Creates missing host dirs | Errors if dir missing |
| Volume options | Appended after colons | Explicit key names |
| tmpfs support | Via --tmpfs flag | Built-in type=tmpfs |

## Auto-Creation Behavior

The most important behavioral difference is how they handle missing source paths:

```bash
# -v creates the host directory if it doesn't exist
podman run --rm -v /home/user/newdir:/data docker.io/library/alpine:latest ls /data
# /home/user/newdir is created automatically

# --mount errors if the source directory doesn't exist
podman run --rm \
  --mount type=bind,source=/home/user/missing,target=/data \
  docker.io/library/alpine:latest ls /data
# Error: /home/user/missing: no such file or directory
```

## Bind Mount Comparison

```bash
# Using -v for a bind mount
podman run -d --name app1 \
  -v /home/user/html:/usr/share/nginx/html:ro,Z \
  docker.io/library/nginx:latest

# Equivalent using --mount
podman run -d --name app2 \
  --mount type=bind,source=/home/user/html,target=/usr/share/nginx/html,readonly,bind-propagation=rprivate \
  docker.io/library/nginx:latest
```

## Named Volume Comparison

```bash
# Using -v for a named volume
podman run -d --name db1 \
  -v pgdata:/var/lib/postgresql/data \
  docker.io/library/postgres:16

# Equivalent using --mount with volume options
podman run -d --name db2 \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  docker.io/library/postgres:16
```

## Volume Driver Options with --mount

The `--mount` flag can pass volume driver options inline:

```bash
# Create and configure a volume inline
podman run -d --name app \
  --mount type=volume,source=nfs-data,target=/data,volume-opt=type=nfs,volume-opt=device=192.168.1.100:/share,volume-opt=o=addr=192.168.1.100 \
  docker.io/library/nginx:latest
```

## When to Use Each

Use `-v` when:
- You want concise, quick commands
- Simple bind mounts or named volumes
- Scripting with short one-liners

Use `--mount` when:
- You need explicit, readable configuration
- Working with tmpfs or complex volume options
- You want strict error handling for missing paths
- Passing volume driver options inline

## Summary

Both `--volume` and `--mount` attach storage to Podman containers. The `-v` flag is concise and auto-creates missing directories, while `--mount` is explicit, more readable, and errors on missing paths. Use `-v` for quick commands and `--mount` for complex configurations or when you need strict path validation. Both support the same underlying mount types and options.
