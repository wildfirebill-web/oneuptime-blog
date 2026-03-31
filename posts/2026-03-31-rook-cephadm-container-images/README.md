# How to Manage Ceph Container Images with cephadm

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, cephadm, Container, Image, Upgrade

Description: Learn how to configure, pin, pull, and upgrade Ceph container images with cephadm for controlled rollouts and airgapped deployments.

---

## Container Images in cephadm

cephadm runs all Ceph daemons as containers. The container image defines the Ceph version running on each daemon. Managing images correctly is essential for controlled upgrades, airgapped environments, and debugging specific Ceph versions.

## Viewing the Current Image

```bash
# Show image used for all daemons
ceph config get global container_image

# Show per-daemon image details
ceph orch ps --format json | python3 -m json.tool | grep image
```

## Official Image Locations

Ceph publishes images to multiple registries:

| Registry | Path | Notes |
|----------|------|-------|
| quay.io | `quay.io/ceph/ceph:v17.2.6` | Official primary |
| Docker Hub | `ceph/ceph:v17.2.6` | Mirror |

```bash
# Check available tags on quay.io
# Use: https://quay.io/repository/ceph/ceph?tab=tags
```

## Pinning a Specific Image

Lock the cluster to a known-good image version:

```bash
# Set a specific image for all cephadm-managed daemons
ceph config set global container_image quay.io/ceph/ceph:v17.2.6

# Verify
ceph config get global container_image
```

## Pulling Images in Advance

Pre-pull container images on all hosts before an upgrade to reduce downtime:

```bash
# Pull image on all managed hosts
ceph cephadm pull-image quay.io/ceph/ceph:v17.2.7 --hosts all

# Pull on specific hosts
ceph cephadm pull-image quay.io/ceph/ceph:v17.2.7 --hosts node1,node2,node3
```

Verify the image is available:

```bash
# SSH to a host and check
ssh root@node1 podman images | grep ceph
```

## Configuring a Private Registry

For airgapped environments, mirror the Ceph image to a private registry:

```bash
# Mirror the image (run on a machine with internet access)
podman pull quay.io/ceph/ceph:v17.2.6
podman tag quay.io/ceph/ceph:v17.2.6 registry.internal.example.com/ceph/ceph:v17.2.6
podman push registry.internal.example.com/ceph/ceph:v17.2.6
```

Configure cephadm to use the private registry:

```bash
# Set custom image URL
ceph config set global container_image registry.internal.example.com/ceph/ceph:v17.2.6

# If authentication is required
ceph cephadm registry-login \
  --registry-url registry.internal.example.com \
  --registry-username myuser \
  --registry-password mypassword
```

## Upgrading the Cluster

Upgrade by setting a new image and triggering the upgrade:

```bash
# Set the target image
ceph orch upgrade start --ceph-version 17.2.7

# Monitor upgrade progress
ceph orch upgrade status

# Watch for daemon restarts
ceph orch ps | grep starting
```

cephadm performs a rolling upgrade: one daemon at a time, waiting for health before proceeding.

## Pausing and Resuming Upgrades

```bash
# Pause the upgrade (stops after current daemon completes)
ceph orch upgrade pause

# Resume the upgrade
ceph orch upgrade resume

# Stop and abort upgrade (reverts to previous image)
ceph orch upgrade stop
```

## Verifying All Daemons Use the New Image

After upgrade, confirm all daemons run the target image:

```bash
# Show image versions across all daemons
ceph versions

# Expected output after complete upgrade
{
  "mon": {"ceph version 17.2.7 ...": 3},
  "mgr": {"ceph version 17.2.7 ...": 2},
  "osd": {"ceph version 17.2.7 ...": 12}
}
```

## Summary

cephadm manages Ceph container images through the `container_image` configuration key and the `ceph orch upgrade` subsystem. Pinning images to specific versions, pre-pulling on all hosts before an upgrade, and configuring private registries for airgapped environments are the core image management workflows. The rolling upgrade system handles version transitions safely, with pause and resume controls giving operators full control over the upgrade pace.
