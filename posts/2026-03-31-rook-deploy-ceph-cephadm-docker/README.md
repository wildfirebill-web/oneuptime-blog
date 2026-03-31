# How to Deploy Ceph Using cephadm with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, cephadm, Docker, Deployment, Storage

Description: Learn how to deploy a Ceph cluster using cephadm with Docker as the container runtime, covering bootstrap, host management, and OSD deployment.

---

## cephadm with Docker

While cephadm uses Podman by default, Docker is fully supported as an alternative container runtime. This is useful on hosts where Podman is not available or when your organization standardizes on Docker. The deployment process is nearly identical to Podman-based deployments, with a few Docker-specific flags.

## Prerequisites

Install Docker on all hosts:

```bash
# Ubuntu/Debian
sudo apt-get install -y docker.io
sudo systemctl enable --now docker

# RHEL/CentOS
sudo dnf install -y docker
sudo systemctl enable --now docker

# Verify Docker
docker --version
docker run hello-world
```

Install cephadm:

```bash
curl --silent --remote-name --location \
  https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
chmod +x cephadm
sudo mv cephadm /usr/local/bin/

sudo cephadm add-repo --release quincy
sudo cephadm install
```

## Bootstrapping with Docker

Pass `--docker` flag to force Docker usage:

```bash
sudo cephadm bootstrap \
  --mon-ip 192.168.1.10 \
  --docker \
  --cluster-network 192.168.2.0/24 \
  --initial-dashboard-user admin \
  --initial-dashboard-password changeme123
```

cephadm detects Docker and configures all subsequent daemon deployments to use it.

## Verifying Docker-Based Daemons

After bootstrap, confirm daemons are running as Docker containers:

```bash
# List running Ceph containers
sudo docker ps --filter label=ceph=true

# Inspect monitor container
sudo docker inspect ceph-<cluster-id>-mon.node1

# View daemon logs
sudo docker logs ceph-<cluster-id>-mon.node1
```

## Adding Hosts

The host addition workflow is identical to Podman:

```bash
# Copy SSH key to additional nodes
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.11

# Add host to cluster
sudo cephadm shell -- ceph orch host add node2 192.168.1.11

# Label the host for OSD placement
sudo cephadm shell -- ceph orch host label add node2 osd
```

## Deploying OSDs

```bash
# List available devices
sudo cephadm shell -- ceph orch device ls --wide

# Deploy OSDs on specific devices
sudo cephadm shell -- ceph orch apply osd --all-available-devices

# Or use a service spec file
cat > osd-spec.yaml << 'EOF'
service_type: osd
service_id: default
placement:
  host_pattern: 'node*'
data_devices:
  all: true
EOF

sudo cephadm shell -- ceph orch apply -i /tmp/osd-spec.yaml
```

## Managing cephadm Configuration

Store cephadm's Docker preference persistently:

```bash
# Verify cephadm container runtime setting
sudo cephadm shell -- ceph config get mgr mgr/cephadm/container_init

# Set Docker as default runtime
sudo cephadm shell -- ceph config set mgr mgr/cephadm/docker true
```

## Pulling Custom Container Images

If you use a private registry or custom Ceph builds:

```bash
# Configure a custom image
sudo cephadm shell -- ceph config set global container_image \
  registry.example.com/ceph/ceph:v17.2.0

# Pull the image on all hosts
sudo cephadm shell -- ceph orch upgrade check
```

## Upgrading the Cluster

cephadm handles rolling upgrades automatically:

```bash
# Start upgrade to a new Ceph version
sudo cephadm shell -- ceph orch upgrade start --ceph-version 17.2.6

# Monitor upgrade progress
sudo cephadm shell -- ceph orch upgrade status
```

## Summary

Deploying Ceph with cephadm and Docker follows the same workflow as Podman-based deployments with the addition of `--docker` during bootstrap. cephadm manages all daemon lifecycle operations through Docker, providing consistent tooling for monitoring logs, running upgrades, and adding capacity. Docker-based deployments are a practical choice for teams with existing Docker infrastructure who want the operational simplicity of cephadm without migrating to Podman.
