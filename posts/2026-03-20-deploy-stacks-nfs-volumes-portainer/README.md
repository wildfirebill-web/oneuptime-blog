# How to Deploy Stacks with Named Volumes and NFS Mounts in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, NFS, Volumes, Storage, Stacks, Infrastructure

Description: Configure Docker stacks with NFS-backed named volumes in Portainer to share persistent storage across multiple containers and hosts in your Docker Swarm cluster.

---

Docker named volumes backed by NFS allow multiple containers — even on different Swarm nodes — to share the same persistent storage. This is essential for stateful services that need shared file access across a cluster. Portainer's stack interface makes NFS volume configuration straightforward.

## When to Use NFS Volumes

- Stateful services in Swarm that may reschedule across nodes
- Shared configuration or data files accessed by multiple services
- Media file storage accessed by multiple web server replicas
- Log directories shared between the application and a log forwarder

## Prerequisites

- An NFS server accessible from all cluster nodes
- NFS client packages installed on host nodes: `apt install nfs-common`

## Step 1: Verify NFS Access

Test NFS connectivity from each node before deploying:

```bash
# Test NFS mount manually
mkdir -p /mnt/test-nfs
mount -t nfs 192.168.1.100:/exports/appdata /mnt/test-nfs
ls /mnt/test-nfs
umount /mnt/test-nfs
```

## Step 2: Define NFS Volumes in Stack YAML

```yaml
# nfs-stack.yml
version: "3.8"

services:
  web:
    image: nginx:alpine
    volumes:
      # Use the NFS-backed named volume
      - shared-media:/usr/share/nginx/html/media:ro
    deploy:
      replicas: 3
    restart: unless-stopped

  media-processor:
    image: my-media-processor:latest
    volumes:
      - shared-media:/data/media
    restart: unless-stopped

volumes:
  shared-media:
    driver: local
    driver_opts:
      type: nfs
      # NFS server IP and export path
      o: "addr=192.168.1.100,nfsvers=4,rw,soft"
      device: ":/exports/shared-media"
```

## Step 3: NFS Volume with Authentication Options

For NFS servers requiring specific mount options:

```yaml
volumes:
  app-data:
    driver: local
    driver_opts:
      type: nfs4
      o: "addr=192.168.1.100,nfsvers=4.1,rw,hard,intr,timeo=600,retrans=2"
      device: ":/exports/app-data"
```

Common NFS mount options:

| Option | Effect |
|---|---|
| `nfsvers=4` | Use NFSv4 |
| `rw` | Read-write |
| `ro` | Read-only |
| `hard` | Retry indefinitely on server failure |
| `soft` | Return error after timeout |
| `noatime` | Skip access time updates (performance) |

## Step 4: Deploy via Portainer

Paste the stack YAML in **Stacks > Add Stack** and click **Deploy the stack**. Portainer creates the NFS-backed volume on first deploy. All containers in the stack that reference `shared-media` will mount the same NFS path.

## Step 5: Verify the Volume

After deployment, check the volume from Portainer's terminal:

```bash
docker volume inspect <stack-name>_shared-media
# Shows the NFS driver options and mount point
```

## Step 6: NFS Performance Considerations

- Use `noatime,nodiratime` to reduce unnecessary NFS writes
- For databases, avoid NFS — use local volumes with placement constraints instead
- For large media files, consider object storage (MinIO) rather than NFS

## Summary

NFS-backed named volumes in Portainer stacks enable shared persistent storage across Swarm nodes without requiring applications to implement distributed storage logic. The volume definition in the stack YAML is self-documenting and reproducible — anyone deploying the stack gets the correct NFS configuration automatically.
