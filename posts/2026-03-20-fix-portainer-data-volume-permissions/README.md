# How to Fix Portainer Data Volume Permission Issues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Docker Volumes, Permissions, Linux, Self-Hosting

Description: Learn how to diagnose and fix permission errors on the Portainer data volume, including correct ownership settings for the BoltDB database and TLS certificate files.

---

Portainer stores its database (`portainer.db`), TLS certificates, and agent keys in a data volume. Incorrect file ownership causes startup failures with cryptic errors like `permission denied` or `failed to open database`.

## Common Symptoms

- Portainer logs show: `bolt: timeout` or `open /data/portainer.db: permission denied`
- Portainer starts but immediately crashes with `failed to create TLS service`
- After a host migration or volume copy, Portainer refuses to start

## Check Current Ownership

```bash
# Inspect ownership of files in the Portainer data volume

docker run --rm -v portainer_data:/data alpine ls -la /data

# You should see something like:
# -rw-------  1 root root  portainer.db
# If owned by a different UID, fix it below
```

## Fix Ownership on Named Volume

Portainer runs as root (UID 0) by default. All files in the data volume must be owned by root:

```bash
# Fix ownership of all files in the data volume
docker run --rm -v portainer_data:/data alpine \
  chown -R root:root /data

# Fix permissions on the database file
docker run --rm -v portainer_data:/data alpine \
  chmod 600 /data/portainer.db
```

## Fix Ownership on Bind Mount

If you use a bind mount (host path) instead of a named volume:

```bash
# The Portainer data directory on the host
sudo chown -R root:root /opt/portainer/data
sudo chmod 700 /opt/portainer/data
sudo chmod 600 /opt/portainer/data/portainer.db
```

## Verify After Fix

```bash
# Start Portainer and check it comes up cleanly
docker start portainer
docker logs portainer --tail 20

# Look for:
# level=info msg="Starting Portainer ..."
# level=info msg="Creating TLS folder in: /data/certs"
# NOT "permission denied" errors
```

## Prevent Issues During Backups

When backing up the Portainer volume, preserve ownership:

```bash
# Correct: use tar with ownership preservation
docker run --rm -v portainer_data:/data alpine \
  tar czpf - /data > portainer-backup.tar.gz

# When restoring, extract with ownership preserved
tar xzpf portainer-backup.tar.gz -C /
```

Using `cp -r` without `-p` loses ownership information and causes permission errors on restore.
