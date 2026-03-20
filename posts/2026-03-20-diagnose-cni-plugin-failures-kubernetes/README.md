# How to Diagnose CNI Plugin Failures During Pod Creation in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, CNI, IPv4, Troubleshooting, Pod Networking, Linux

Description: Diagnose common CNI plugin failures that prevent Kubernetes pods from getting IPv4 addresses and starting successfully.

CNI (Container Network Interface) failures during pod creation prevent pods from getting IP addresses, leaving them stuck in `ContainerCreating`. This guide covers systematic diagnosis.

## Step 1: Identify the Failure

```bash
# Find pods stuck in ContainerCreating
kubectl get pods --all-namespaces | grep ContainerCreating

# Get the detailed error
kubectl describe pod <pod-name> -n <namespace>

# Common CNI-related events:
# "Failed to create pod sandbox: rpc error: ... failed to setup network"
# "network: failed to allocate for range 0: no IP addresses available"
# "Error adding network: failed to find CNI config"
```

## Step 2: Check CNI Plugin Pods

```bash
# For Calico
kubectl get pods -n kube-system -l k8s-app=calico-node
kubectl get pods -n calico-system

# For Flannel
kubectl get pods -n kube-flannel -l app=flannel

# For Cilium
kubectl get pods -n kube-system -l k8s-app=cilium

# Check logs of the CNI plugin pod on the node where the failing pod was scheduled
kubectl logs -n kube-system <cni-pod-on-node>
```

## Step 3: Check CNI Configuration Files

```bash
# SSH to the node where the pod creation failed
# CNI config files must exist and be valid JSON
ls /etc/cni/net.d/
# Expected: 10-calico.conflist or 10-flannel.conflist etc.

cat /etc/cni/net.d/10-calico.conflist

# Validate JSON syntax
python3 -m json.tool /etc/cni/net.d/10-flannel.conflist
# "No JSON object could be decoded" = invalid config
```

## Step 4: Check CNI Binary Files

```bash
# CNI binaries must exist on every node
ls /opt/cni/bin/
# Expected: flannel, calico, calico-ipam, host-local, loopback, etc.

# If missing, reinstall the CNI plugin
# For Flannel:
kubectl delete -f kube-flannel.yml && kubectl apply -f kube-flannel.yml
```

## Step 5: Check the kubelet Logs

```bash
# kubelet invokes the CNI plugin and logs failures
sudo journalctl -u kubelet -f | grep -i "cni\|network\|failed"

# Typical kubelet errors:
# "CNI failed to retrieve network namespace path"
# "Error occurred while creating pod, err: failed to ensure pod"
```

## Step 6: Test the CNI Plugin Directly

```bash
# Get network namespace of a failing container
CONTAINER_ID=$(docker ps -a | grep k8s_POD | head -1 | awk '{print $1}')
NETNS=$(docker inspect $CONTAINER_ID --format '{{.State.Pid}}')

# Run the CNI plugin manually (replace with your CNI binary path)
CNI_COMMAND=ADD \
CNI_CONTAINERID=$CONTAINER_ID \
CNI_NETNS=/proc/$NETNS/ns/net \
CNI_IFNAME=eth0 \
CNI_PATH=/opt/cni/bin \
/opt/cni/bin/flannel < /etc/cni/net.d/10-flannel.conflist
```

## Step 7: Check Node IP Pool Exhaustion

```bash
# For Calico, check if IP pool is exhausted
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config calicoctl ipam show --summary

# For Flannel, check the subnet file on the node
cat /run/flannel/subnet.env
```

## Common CNI Failures and Fixes

| Error | Cause | Fix |
|---|---|---|
| `no IP addresses available` | IP pool exhausted | Expand IP pool or release leaked IPs |
| `failed to find CNI config` | Missing config file | Reinstall CNI plugin |
| `no CNI binary found` | Missing binary | Copy CNI binary to /opt/cni/bin |
| `failed to create network` | CNI pod not running | Restart CNI DaemonSet |
| `pod CIDR not set` | Node not initialized | Check controller-manager --allocate-node-cidrs |

## Resetting CNI State on a Node

```bash
# For emergency node CNI reset (use carefully)
sudo systemctl stop kubelet
sudo rm -rf /var/lib/cni/
sudo systemctl start kubelet
```

After a fresh start, the CNI plugin will reinitialize and re-allocate IPs for surviving pods.
