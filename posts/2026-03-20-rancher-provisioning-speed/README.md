# How to Optimize Cluster Provisioning Speed in Rancher - Speed

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Provisioning, Performance, RKE2

Description: Speed up Kubernetes cluster provisioning in Rancher by optimizing node images, container registry mirrors, and provisioning configuration for faster deployments.

## Introduction

Slow cluster provisioning frustrates operators and slows down development workflows. Rancher cluster provisioning time can range from 5 minutes to 30+ minutes depending on configuration. This guide covers the key optimizations that can cut provisioning time by 50-70%.

## Prerequisites

- Rancher with cloud provisioner access (AWS, Azure, GCP, vSphere)
- Container registry mirror available
- Pre-baked node images (optional but recommended)

## Step 1: Use Pre-Baked Node Images

```bash
# Create a base AMI with Kubernetes dependencies pre-installed

# This eliminates package installation time during provisioning

# On a base EC2 instance, run:
# Install container runtime
curl -fsSL https://get.docker.com | sh
systemctl enable docker

# Pre-pull RKE2 images
systemctl enable rke2-server
rke2-install.sh  # Download installer

# Pre-pull common Kubernetes images
ctr images pull registry.k8s.io/pause:3.9
ctr images pull registry.k8s.io/coredns/coredns:v1.10.1
ctr images pull registry.k8s.io/etcd:3.5.9-0
ctr images pull registry.k8s.io/kube-apiserver:v1.28.0
ctr images pull registry.k8s.io/kube-controller-manager:v1.28.0
ctr images pull registry.k8s.io/kube-scheduler:v1.28.0
ctr images pull registry.k8s.io/kube-proxy:v1.28.0

# Create AMI from this instance
aws ec2 create-image \
  --instance-id i-xxxxxxxx \
  --name "rke2-k8s-1.28-base-$(date +%Y%m%d)" \
  --description "Pre-baked RKE2 Kubernetes 1.28 node"
```

## Step 2: Configure Registry Mirror for Faster Pulls

```yaml
# registries.yaml - Pre-configure registry mirror in node image
# Place at /etc/rancher/rke2/registries.yaml before AMI creation

mirrors:
  docker.io:
    endpoint:
      - "https://mirror.registry.internal"
  registry.k8s.io:
    endpoint:
      - "https://mirror.registry.internal"
  ghcr.io:
    endpoint:
      - "https://mirror.registry.internal"

configs:
  "mirror.registry.internal":
    tls:
      ca_file: /etc/ssl/certs/internal-ca.crt
```

## Step 3: Optimize Rancher Machine Config (AWS)

```yaml
# machine-config-fast.yaml - Optimized AWS machine configuration
apiVersion: rke-machine-config.cattle.io/v1
kind: Amazonec2Config
metadata:
  name: fast-worker-config
  namespace: fleet-default
spec:
  # Use fast instance types with NVMe SSD
  instanceType: m6i.xlarge

  # Use pre-baked AMI
  ami: ami-xxxxxxxxxxxxxxxxx

  # Use latest generation for faster network
  iamInstanceProfile: K8sWorkerRole

  # Pre-attached security groups
  securityGroup:
    - k8s-workers-sg

  # Use GP3 SSD for root volume
  rootSize: 50
  volumeType: gp3
  iops: 3000
  throughput: 125

  # Use placement group for low-latency inter-node communication
  placementGroup: k8s-cluster-pg

  # Userdata - minimal since using pre-baked AMI
  userdata: |
    #!/bin/bash
    # Set hostname
    hostnamectl set-hostname $(curl -s http://169.254.169.254/latest/meta-data/local-hostname)
```

## Step 4: Parallelize Node Provisioning

```yaml
# cluster-template-fast.yaml - Parallel node provisioning
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: production-cluster
  namespace: fleet-default
spec:
  rkeConfig:
    nodePools:
      # Control plane nodes
      - name: control-plane
        quantity: 3
        roles:
          - controlplane
          - etcd
        machineConfigRef:
          kind: Amazonec2Config
          name: fast-cp-config

      # Worker nodes - provision all in parallel
      - name: workers
        quantity: 10
        roles:
          - worker
        machineConfigRef:
          kind: Amazonec2Config
          name: fast-worker-config
        # No drain on upgrade for faster scaling
        rollingUpdate:
          maxUnavailable: 2
          maxSurge: 2
```

## Step 5: Pre-stage Rancher Agent Images

```bash
# On pre-baked AMI, pull the Rancher agent images
# This eliminates the agent pull time

# Check which Rancher agent images are needed
# (Version-specific, update for your Rancher version)
docker pull rancher/rancher-agent:v2.8.0

# For RKE2, pre-pull system images
rke2 images list | while read image; do
  ctr images pull $image
done

# Save images as tarball for air-gapped environments
rke2 images save > /var/lib/rancher/rke2/agent/images/rke2-images.tar
```

## Step 6: Optimize DNS Resolution

```bash
# Slow DNS can significantly slow provisioning
# Ensure proper DNS search domains in cloud provider

# Test DNS resolution speed on a node
time nslookup rancher.example.com

# Configure faster DNS on nodes
cat >> /etc/resolv.conf << EOF
options timeout:1 attempts:3 rotate
EOF

# For EC2 instances, use Route53 Resolver
# Ensure your VPC has DNS hostnames and DNS resolution enabled
aws ec2 modify-vpc-attribute --vpc-id vpc-xxxxx --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id vpc-xxxxx --enable-dns-support
```

## Step 7: Monitor Provisioning Time

```bash
# Track provisioning time for clusters
kubectl get clusters.provisioning.cattle.io \
  -n fleet-default \
  -o json | jq -r '
    .items[] |
    {
      name: .metadata.name,
      created: .metadata.creationTimestamp,
      ready: (.status.conditions[] | select(.type=="Ready") | .lastTransitionTime)
    }'

# Alert on slow provisioning
# Create a PrometheusRule to alert if provisioning takes > 15 minutes
```

## Conclusion

Cluster provisioning speed in Rancher is primarily determined by three factors: node image preparation time, container image pull speed, and Rancher agent connection latency. By using pre-baked AMIs with Kubernetes binaries and images pre-loaded, combined with internal registry mirrors, you can reduce provisioning time from 20+ minutes to under 5 minutes. For organizations that frequently provision new clusters-such as in CI/CD workflows or auto-scaling scenarios-these optimizations are essential for operational efficiency.
