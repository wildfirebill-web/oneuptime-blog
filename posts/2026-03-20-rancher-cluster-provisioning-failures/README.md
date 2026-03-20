# How to Troubleshoot Cluster Provisioning Failures in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Troubleshooting, Provisioning

Description: Diagnose and fix cluster provisioning failures in Rancher, covering RKE2, K3s, and hosted provider issues with practical debugging steps.

## Introduction

Cluster provisioning failures in Rancher can occur at multiple stages: node bootstrapping, Kubernetes component startup, or CNI/CSI installation. This guide covers the most common failure scenarios across RKE2, K3s, and cloud-hosted clusters and how to resolve them.

## Step 1: Check Provisioning Status in the UI

Navigate to **Cluster Management** in Rancher and look for clusters stuck in:

- **Provisioning** — infrastructure is being created but Kubernetes hasn't started yet.
- **Waiting** — Rancher is waiting for nodes to register.
- **Updating** — components are being applied but the process stalled.

Click on the cluster name and look at the **Conditions** tab for specific error messages.

## Step 2: Examine Provisioning Logs

```bash
# Check the Rancher server logs for provisioning errors
kubectl logs -n cattle-system -l app=rancher --tail=300 | grep -i "provision\|error\|fail"

# For RKE2/K3s provisioning, check the provisioning controller
kubectl logs -n cattle-provisioning-capi-system \
  -l control-plane=controller-manager --tail=200

# Check provisioning machine resources
kubectl get machines -n fleet-default
kubectl describe machine -n fleet-default <machine-name>
```

## Step 3: Check Node Bootstrap Logs

For RKE2 nodes, SSH into the node and check the bootstrap process:

```bash
# On the provisioned node
# Check cloud-init / user-data execution
sudo cat /var/log/cloud-init-output.log

# Check RKE2 service status
sudo systemctl status rke2-server   # for server nodes
sudo systemctl status rke2-agent    # for agent nodes

# Stream RKE2 logs
sudo journalctl -u rke2-server -f --no-pager

# Check RKE2 agent registration log
sudo cat /var/lib/rancher/rke2/agent/logs/kubelet.log
```

## Step 4: Common Provisioning Failure Causes

### Insufficient Node Resources

```bash
# Check node resources before provisioning
# Minimum for RKE2: 2 vCPU, 4 GB RAM, 20 GB disk

# On the node:
free -h       # Check available memory
df -h /       # Check root disk space
nproc         # Check CPU count
```

### Node Registration Timeout

```bash
# The node must reach the Rancher server within the registration timeout
# Check if the node can reach Rancher:
curl -k https://<rancher-url>/ping

# Check the registration token on the node
sudo cat /var/lib/rancher/rke2/server/token
```

### Port Conflicts

```bash
# RKE2 requires specific ports to be free
# Check for conflicts on the node
sudo ss -tlnp | grep -E '6443|9345|10250|2379|2380'

# Common conflict: another Kubernetes installation
sudo which k3s kubectl   # Should not exist on a fresh node
```

## Step 5: Cloud Provider Provisioning Issues

### AWS EC2 Provisioning

```bash
# Check Rancher's AWS credentials
kubectl get secret -n cattle-global-data aws-creds -o json \
  | jq '.data | map_values(@base64d)'

# Verify the IAM role has required permissions (EC2, VPC, IAM)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::ACCOUNT:user/rancher \
  --action-names ec2:RunInstances ec2:DescribeInstances
```

### vSphere Provisioning

```bash
# Check vSphere credentials secret
kubectl get secret -n cattle-global-data vsphere-creds -o json \
  | jq -r '.data | to_entries[] | "\(.key): \(.value | @base64d)"'

# Common issues:
# - Invalid datacenter/datastore/network paths
# - Template VM not found or powered on
# - Insufficient vSphere permissions
```

## Step 6: Fix a Stalled Provisioning

```bash
# Force-delete the stalled cluster resource (USE WITH CAUTION)
# First, remove the finalizer
kubectl patch cluster.provisioning.cattle.io -n fleet-default <cluster-name> \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

# Then delete
kubectl delete cluster.provisioning.cattle.io -n fleet-default <cluster-name>
```

## Step 7: Review Machine Pool Events

```bash
# List all CAPI machine resources
kubectl get machineset,machine,machinedeployment -n fleet-default

# Get events for a specific machine
kubectl get events -n fleet-default \
  --field-selector reason!=Pulling,reason!=Pulled \
  | grep <machine-name>
```

## Conclusion

Cluster provisioning failures in Rancher require examining logs at multiple levels: the Rancher server, the CAPI provisioning controller, and the individual nodes. The most frequent causes are insufficient node resources, network connectivity issues between nodes and Rancher, port conflicts, and incorrect cloud provider credentials. Addressing these systematically will resolve the vast majority of provisioning failures.
