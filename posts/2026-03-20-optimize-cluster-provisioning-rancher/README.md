# How to Optimize Cluster Provisioning Speed in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Provisioning, Performance, Kubernetes, Node Templates, Automation

Description: Speed up Rancher cluster provisioning by pre-caching images, optimizing node templates, using custom machine drivers, and parallelizing node bootstrapping.

## Introduction

Cluster provisioning time in Rancher depends on node boot time, image pull time, and Kubernetes component startup. In cloud environments, a new cluster can take 15-30 minutes with default settings. These optimizations can reduce that to 5-10 minutes.

## Step 1: Use Pre-Configured Node Images

Pre-bake Kubernetes images into your machine image (AMI, GCE Image) to eliminate image pull time during provisioning:

```bash
# On a base node, pre-pull all RKE2 images
systemctl start rke2-agent

# Run RKE2 to pull all container images
sudo rke2 server --cluster-init &
sleep 60
sudo rke2-killall.sh

# Package images into the node image for your cloud provider
# Now all new nodes start with images pre-cached
```

## Step 2: Optimize Node Template Configuration

```yaml
# Rancher Node Template (AWS example)
amazonec2Config:
  instanceType: t3.large
  ami: ami-PREBAKED-IMAGE      # AMI with Kubernetes images pre-loaded
  rootSize: "50"
  volumeType: gp3              # gp3 has better baseline performance than gp2
  iops: "3000"
  throughput: "125"
  spotPrice: ""                # On-demand for production stability

# Use user-data to configure nodes faster
userdata: |
  #!/bin/bash
  # Disable swap (required for Kubernetes)
  swapoff -a
  sed -i '/swap/d' /etc/fstab
  # Configure sysctl settings required by Kubernetes
  sysctl -w net.ipv4.ip_forward=1
  sysctl -w net.bridge.bridge-nf-call-iptables=1
```

## Step 3: Increase Provisioning Parallelism

For large clusters, Rancher provisions nodes sequentially by default. Enable parallel provisioning:

```bash
# Set CATTLE_NEW_NODE_PER_MINUTE to allow faster provisioning
kubectl set env deployment/rancher \
  -n cattle-system \
  CATTLE_NEW_NODE_PER_MINUTE=20    # Allow 20 nodes per minute (default 5)
```

## Step 4: Use RKE2 with Embedded Component Cache

RKE2 bundles all components as a self-contained archive, eliminating download time:

```bash
# On the Rancher UI when creating a cluster:
# Select "RKE2/K3s" as the cluster type
# Configure the RKE2 version
# RKE2 uses airgap bundles that eliminate network-dependent image pulls
```

## Step 5: Pre-Configure etcd Snapshots Schedule

Configure etcd snapshots after cluster creation to avoid post-provision pauses:

```yaml
# RKE2 cluster config
etcd:
  snapshot:
    schedule: "0 */6 * * *"    # Every 6 hours
    retention: 5
```

## Step 6: Monitor Provisioning Progress

```bash
# Watch cluster provisioning events
kubectl get events -n cattle-system \
  --field-selector reason=ProvisioningSuccessful \
  --watch

# View Rancher provisioning logs
kubectl logs -n cattle-system rancher-xxxxx | grep -E "provision|cluster" | tail -50
```

## Conclusion

Cluster provisioning speed depends primarily on image pull time and node bootstrap time. Pre-baked AMIs with Kubernetes images are the single highest-impact optimization, reducing provisioning time by 5-10 minutes per cluster. Combined with parallel node provisioning and RKE2's self-contained bundles, you can provision a 10-node cluster in under 8 minutes.
