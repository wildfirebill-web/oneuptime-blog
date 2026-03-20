# How to Configure Volume Drivers in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Volumes, Storage, DevOps

Description: Learn how to configure and use different Docker volume drivers in Portainer for local, NFS, cloud, and distributed storage backends.

## Introduction

Docker's volume driver system allows you to use different storage backends for container volumes — from local filesystem to NFS, cloud storage (AWS EBS, Azure File), and distributed storage systems (Ceph, GlusterFS). Portainer exposes volume driver configuration when creating volumes, giving you flexibility to connect containers to any storage system.

## Prerequisites

- Portainer installed with a connected Docker environment
- Appropriate volume driver plugin installed (for non-local drivers)

## Built-in Volume Drivers

### local Driver (Default)

The default driver stores volumes on the local host filesystem:

```yaml
# Default: local driver
volumes:
  mydata:
    driver: local  # Optional — local is the default

# Local driver with custom options (creates a bind mount-backed volume):
volumes:
  config_data:
    driver: local
    driver_opts:
      type: none       # Don't create a filesystem
      o: bind          # Bind mount mode
      device: /host/path/to/data  # Host directory
```

### local Driver with NFS

```yaml
volumes:
  nfs_data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=192.168.1.10,rw,nfsvers=4"
      device: ":/exports/data"
```

### local Driver with CIFS/SMB

```yaml
volumes:
  smb_data:
    driver: local
    driver_opts:
      type: cifs
      o: "addr=server,username=user,password=pass,vers=3.0"
      device: "//server/share"
```

### local Driver with tmpfs (Memory)

```yaml
# tmpfs: in-memory storage (lost on container restart)
volumes:
  cache:
    driver: local
    driver_opts:
      type: tmpfs
      device: tmpfs
      o: "size=256m"  # Max 256 MB
```

## Installing Third-Party Volume Driver Plugins

For cloud and distributed storage, install driver plugins:

```bash
# AWS EBS driver:
docker plugin install rexray/ebs:latest \
  EBS_ACCESSKEY=AKIAXXXXXXXX \
  EBS_SECRETKEY=xxxxxxxxxx \
  EBS_REGION=us-east-1

# Azure File driver:
docker plugin install rexray/azureud:latest

# GlusterFS driver:
docker plugin install surnet/docker-volume-glusterfs:latest

# Flocker (Cluster Storage):
docker plugin install clusterhq/flocker-docker-plugin

# Portworx:
docker plugin install portworx/pxdev

# List installed plugins:
docker plugin ls
```

## Step 1: Configure Volume Driver in Portainer

1. Navigate to **Volumes > Add volume**.
2. Enter a volume name.
3. Under **Driver**, select from installed drivers.
4. Under **Driver options**, enter key-value pairs for the driver.

```
Name:    ebs-production-data
Driver:  rexray/ebs
Options:
  size: 100     (100 GB EBS volume)
  volumetype: gp3
  iops: 3000
```

## Step 2: AWS EBS Volume Driver

For containerized applications on EC2:

```bash
# Install REX-Ray EBS plugin:
docker plugin install rexray/ebs:latest \
  EBS_ACCESSKEY=${AWS_ACCESS_KEY} \
  EBS_SECRETKEY=${AWS_SECRET_KEY} \
  EBS_REGION=us-east-1 \
  --grant-all-permissions
```

```yaml
# docker-compose.yml with EBS volumes:
volumes:
  db_data:
    driver: rexray/ebs
    driver_opts:
      size: "50"         # 50 GB EBS volume
      volumetype: "gp3"  # General Purpose SSD
      iops: "3000"
      encrypted: "true"
```

## Step 3: Azure File Driver

```bash
# Install Azure volume plugin:
docker plugin install rexray/azureud:latest \
  AZURE_SUBSCRIPTIONID=${AZURE_SUBSCRIPTION_ID} \
  AZURE_TENANTID=${AZURE_TENANT_ID} \
  AZURE_CLIENTID=${AZURE_CLIENT_ID} \
  AZURE_CLIENTSECRET=${AZURE_CLIENT_SECRET} \
  --grant-all-permissions
```

Or use Azure Files directly with CIFS:

```yaml
volumes:
  azure_files:
    driver: local
    driver_opts:
      type: cifs
      o: "addr=${STORAGE_ACCOUNT}.file.core.windows.net,username=${STORAGE_ACCOUNT},password=${STORAGE_KEY},vers=3.0,serverino"
      device: "//${STORAGE_ACCOUNT}.file.core.windows.net/${SHARE_NAME}"
```

## Step 4: GlusterFS Volume Driver

For distributed storage across multiple hosts:

```bash
# Install GlusterFS plugin:
docker plugin install surnet/docker-volume-glusterfs:latest \
  --grant-all-permissions
```

```yaml
volumes:
  gluster_data:
    driver: surnet/docker-volume-glusterfs
    driver_opts:
      servers: "gluster1:gluster2:gluster3"
      volname: "gv0"
```

## Step 5: Portworx Volume Driver

For enterprise container storage:

```yaml
volumes:
  px_database:
    driver: pxd
    driver_opts:
      size: "50G"
      repl: "3"        # Replicate across 3 nodes
      priority_io: "high"
      label: "env:production"
```

## Step 6: Volume Driver in Portainer UI

When creating a volume in Portainer:

1. Navigate to **Volumes > Add volume**.
2. **Driver** dropdown shows all installed volume driver plugins.
3. Select the desired driver.
4. Add driver options as key-value pairs.
5. Create the volume.

## Step 7: Verify Driver Installation

```bash
# List all installed volume driver plugins:
docker plugin ls

# Output:
ID            NAME            DESCRIPTION                      ENABLED
abc123        rexray/ebs      REX-Ray for Amazon EBS           true
def456        local           Built-in local volume driver     true

# Test a driver:
docker volume create --driver rexray/ebs --opt size=1 test-ebs-volume
docker volume inspect test-ebs-volume
docker volume rm test-ebs-volume
```

## Step 8: Volume Driver Selection Guide

```
Use Case                    → Recommended Driver
Single host, local data     → local (default)
Multiple hosts, shared data → NFS (local with NFS opts)
Windows file shares         → local with CIFS opts
AWS EC2, persistent         → rexray/ebs or EFS
Azure VMs                   → Azure File (CIFS) or Azure Disk
GCP                         → GCE Persistent Disk driver
Multi-cloud, HA             → Portworx, Rook/Ceph
Edge devices                → local (simple, reliable)
Development                 → local or tmpfs (fast)
```

## Conclusion

Docker volume drivers in Portainer unlock a wide range of storage backends for containers. Start with the built-in `local` driver (which supports NFS, CIFS, and tmpfs via options), and install third-party plugins for cloud-native storage like EBS or Azure File. Choosing the right driver for your infrastructure ensures containers have access to the right storage with appropriate performance, durability, and sharing characteristics.
