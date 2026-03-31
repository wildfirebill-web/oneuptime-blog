# How to Configure Ceph for Docker Swarm Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Docker Swarm, Persistent Storage, NFS, Volume

Description: Provide persistent shared storage to Docker Swarm services using Ceph RBD and CephFS, enabling stateful workloads across Swarm nodes.

---

Docker Swarm schedules services across multiple nodes but does not natively handle persistent storage. Ceph provides the shared storage layer that Swarm services need to maintain state regardless of which node a container is scheduled on.

## Architecture Overview

For Docker Swarm with Ceph:

- **CephFS** - for services that need shared read-write storage (NFS-like access)
- **Ceph RBD** - for single-writer services that need high-performance block storage
- **Ceph RGW** - for services that store and retrieve objects via S3

## Mount CephFS on All Swarm Nodes

CephFS must be mounted on all Swarm manager and worker nodes so that any node can host a service with persistent storage:

```bash
# Run on every Swarm node
apt-get install -y ceph-common ceph-fuse

# Copy Ceph config and keyring to each node
scp /etc/ceph/ceph.conf swarm-node2:/etc/ceph/
scp /etc/ceph/ceph.client.admin.keyring swarm-node2:/etc/ceph/

# Mount on each node
mkdir -p /mnt/ceph

ceph-fuse /mnt/ceph \
  --name client.admin \
  -o allow_other

# Persist with fstab
echo "none /mnt/ceph fuse.ceph ceph.id=admin,ceph.conf=/etc/ceph/ceph.conf,_netdev,allow_other 0 0" >> /etc/fstab
```

## Create Docker Volumes on Each Node

Create a Docker local volume pointing to the CephFS mount:

```bash
# Run on every node
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/mnt/ceph \
  --opt o=bind \
  cephfs
```

## Deploy a Swarm Service with Ceph Storage

Create a service that uses the CephFS volume:

```yaml
version: "3.9"
services:
  app:
    image: myapp:latest
    volumes:
      - type: volume
        source: cephfs
        target: /data
        volume:
          nocopy: true
    deploy:
      replicas: 3
      placement:
        constraints:
          - node.role == worker
      restart_policy:
        condition: on-failure

volumes:
  cephfs:
    external: true
```

Deploy to Swarm:

```bash
docker stack deploy -c docker-compose.yml myapp
```

## Use RBD for High-Performance Services

For single-instance services that need high I/O, use RBD with node pinning:

```yaml
services:
  postgres:
    image: postgres:15
    volumes:
      - /mnt/rbd/postgres:/var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.has_rbd == true
```

Label the node that has the RBD device mounted:

```bash
docker node update --label-add has_rbd=true swarm-node1
```

## Monitor Swarm Service Health

Check that services are running with persistent volumes:

```bash
docker service ls
docker service ps myapp_app --no-trunc
```

Verify data persists after service restart:

```bash
docker service update --force myapp_app
docker service ps myapp_app
```

## Summary

Docker Swarm achieves persistent storage with Ceph by mounting CephFS on all nodes and creating Docker local volumes pointing to those mounts. Services referencing external volumes will have their data available on any node the container is scheduled on. Use placement constraints to pin high-I/O RBD workloads to the specific node with the RBD device mapped.
