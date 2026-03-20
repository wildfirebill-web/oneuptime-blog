# How to Configure RKE2 Server Configuration File

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: RKE2, Kubernetes, Configuration, Server, Rancher

Description: A comprehensive reference guide to all available options in the RKE2 server configuration file for customizing your Kubernetes control plane.

The RKE2 server configuration file (`/etc/rancher/rke2/config.yaml`) is the primary way to configure all aspects of the RKE2 control plane. Understanding all available options allows you to precisely tune your cluster for your specific requirements. This guide serves as a comprehensive reference for the server configuration file.

## Configuration File Location and Format

```bash
# Configuration file path

/etc/rancher/rke2/config.yaml

# The file uses YAML format
# All CLI flags can be expressed as YAML keys by:
# 1. Removing the leading --
# 2. Replacing hyphens with hyphens (they stay the same in YAML)
# Example: --tls-san becomes tls-san

# Verify the configuration
sudo rke2 server --help | grep -A2 "config"
```

## Complete Server Configuration Reference

```yaml
# /etc/rancher/rke2/config.yaml - Complete reference

# =====================
# CLUSTER JOINING
# =====================

# URL of existing server to join (for additional servers)
# server: https://first-server:9345

# Token for cluster authentication
# token: K10xxxxx

# Token file (alternative to token:)
# token-file: /path/to/token

# =====================
# NETWORKING
# =====================

# IP address to advertise to other nodes
# Defaults to first IP of the node
# node-ip: 10.0.0.10

# Optional additional IP addresses
# node-external-ip: 203.0.113.10

# Cluster CIDR for pod networking
cluster-cidr: 10.42.0.0/16

# Service CIDR for services
service-cidr: 10.43.0.0/16

# Cluster DNS server address
cluster-dns: 10.43.0.10

# Cluster domain
cluster-domain: cluster.local

# =====================
# TLS / CERTIFICATES
# =====================

# Additional SANs for the API server TLS certificate
tls-san:
  - "api.example.com"
  - "10.0.0.10"

# =====================
# CNI / NETWORKING
# =====================

# CNI plugin: canal, calico, cilium, or none
cni: canal

# Flannel backend: vxlan, wireguard-native, host-gw, or none
# flannel-backend: vxlan

# Disable kube-proxy (for CNI plugins that replace it)
# disable-kube-proxy: false

# Egress selector mode: agent, cluster, pod, disabled
# egress-selector-mode: agent

# =====================
# RUNTIME
# =====================

# Container runtime endpoint
# container-runtime-endpoint: ""

# Private registry for system images
# system-default-registry: registry.example.com

# SNI for API server TLS
# api-server-service-cidr: ""

# =====================
# ETCD
# =====================

# Disable embedded etcd (use external)
# disable-etcd: false

# External datastore endpoint
# datastore-endpoint: ""
# datastore-cafile: ""
# datastore-certfile: ""
# datastore-keyfile: ""

# Etcd snapshot configuration
etcd-snapshot-schedule-cron: "0 */6 * * *"
etcd-snapshot-retention: 10
etcd-snapshot-dir: /var/lib/rancher/rke2/server/db/snapshots

# S3 snapshot storage
# etcd-s3: false
# etcd-s3-endpoint: s3.amazonaws.com
# etcd-s3-region: ""
# etcd-s3-bucket: ""
# etcd-s3-folder: ""
# etcd-s3-access-key: ""
# etcd-s3-secret-key: ""
# etcd-s3-timeout: 5m
# etcd-s3-skip-ssl-verify: false

# =====================
# KUBERNETES COMPONENTS
# =====================

# API server extra arguments
kube-apiserver-arg:
  - "anonymous-auth=false"
  - "audit-log-path=/var/log/kubernetes/audit.log"

# Controller manager extra arguments
kube-controller-manager-arg:
  - "bind-address=127.0.0.1"

# Scheduler extra arguments
kube-scheduler-arg:
  - "bind-address=127.0.0.1"

# Kubelet extra arguments
kubelet-arg:
  - "anonymous-auth=false"
  - "protect-kernel-defaults=true"
  - "max-pods=110"

# Kube-proxy extra arguments
kube-proxy-arg:
  - "proxy-mode=iptables"

# Etcd extra arguments
etcd-arg:
  - "heartbeat-interval=300"
  - "election-timeout=3000"

# Cloud controller manager extra arguments
# cloud-controller-manager-arg: []

# =====================
# NODE CONFIGURATION
# =====================

# Custom node name (defaults to hostname)
# node-name: my-server-01

# Additional node labels
node-label:
  - "environment=production"

# Node taints
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"

# =====================
# SECURITY
# =====================

# Security hardening profile: cis-1.23
# profile: cis-1.23

# Pod Security Admission config file
# pod-security-admission-config-file: /etc/rancher/rke2/psa.yaml

# =====================
# DISABLE COMPONENTS
# =====================

# Disable specific built-in components
disable:
  # - rke2-coredns
  # - rke2-ingress-nginx
  # - rke2-metrics-server

# =====================
# KUBECONFIG
# =====================

# Kubeconfig file path
# write-kubeconfig: /etc/rancher/rke2/rke2.yaml

# Kubeconfig file permissions
write-kubeconfig-mode: "0644"

# =====================
# CLOUD PROVIDER
# =====================

# Cloud provider name (aws, azure, gce, etc.)
# cloud-provider-name: ""

# Cloud provider configuration file
# cloud-provider-config: /etc/rancher/rke2/cloud-provider.yaml

# =====================
# SELINUX AND FEATURES
# =====================

# Enable SELinux for containers
# selinux: false

# =====================
# IMAGES AND REGISTRY
# =====================

# Private registry configuration file
# (configured in /etc/rancher/rke2/registries.yaml)

# Container image prefix for all images
# image-credential-provider-config: ""
# image-credential-provider-bin-dir: ""
```

## Apply Configuration Changes

```bash
# After modifying config.yaml, restart RKE2 to apply changes
sudo systemctl restart rke2-server

# Verify the configuration was applied
sudo journalctl -u rke2-server | tail -20

# Check that components have the expected arguments
sudo ps aux | grep kube-apiserver | tr ' ' '\n' | \
  grep -E "audit|anonymous|admission"

# Verify cluster status after restart
kubectl get nodes
kubectl get pods -n kube-system | grep -v Running
```

## Conclusion

The RKE2 server configuration file provides a comprehensive, YAML-based interface to configure every aspect of your Kubernetes control plane. By understanding the available options, you can precisely tune your cluster for security (CIS profile, SELinux), performance (etcd arguments, kubelet settings), and operational requirements (snapshot schedules, registry mirrors). Always test configuration changes in a non-production environment and take an etcd backup before applying changes to production clusters.
