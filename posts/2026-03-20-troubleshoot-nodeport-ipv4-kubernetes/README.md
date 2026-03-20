# How to Troubleshoot NodePort Service IPv4 Accessibility in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, NodePort, IPv4, Troubleshooting, Networking, kube-proxy

Description: Diagnose and fix NodePort service accessibility issues where external clients cannot reach Kubernetes services via node IPv4 addresses and high ports.

NodePort services expose a service on every node's IPv4 address on a port between 30000-32767. Accessibility failures usually stem from firewall rules, kube-proxy issues, or incorrect pod health.

## Step 1: Verify the NodePort Assignment

```bash
# Check what NodePort was assigned
kubectl get svc my-nodeport-service
# NAME                TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
# my-nodeport-service NodePort   10.96.45.123   <none>        80:31234/TCP     5m

# The NodePort is 31234 — accessible on any node IP at port 31234
```

## Step 2: Test from Within the Cluster First

```bash
# Get a node IP
kubectl get nodes -o wide | awk '{print $6}' | tail -n +2

# From a pod inside the cluster, test the NodePort
kubectl run debug --image=alpine --restart=Never -- sleep 3600
kubectl exec debug -- wget -qO- http://<NODE_IP>:31234
```

## Step 3: Test from External Network

```bash
# From a machine outside the cluster
curl http://<NODE_IP>:31234

# If timeout, check firewall first (step 4)
# If connection refused, check endpoints (step 5)
```

## Step 4: Check Host Firewall on Nodes

```bash
# SSH to a node and check if the port is open
sudo iptables -L INPUT -n -v | grep 31234

# If the firewall is blocking, add a rule
sudo iptables -A INPUT -p tcp --dport 31234 -j ACCEPT

# For UFW:
sudo ufw allow 31234/tcp

# For firewalld:
sudo firewall-cmd --permanent --add-port=31234/tcp
sudo firewall-cmd --reload
```

## Step 5: Verify Endpoints are Ready

```bash
# Check endpoints
kubectl get endpoints my-nodeport-service
# NAME                  ENDPOINTS           AGE
# my-nodeport-service   10.244.1.5:8080    5m

# If <none>, the service selector doesn't match running pods
kubectl get pods -l <service-selector-labels> -n <namespace>
```

## Step 6: Check kube-proxy iptables Rules

```bash
# On a node, view the NodePort iptables rule
sudo iptables -t nat -L KUBE-NODEPORTS -n
# Look for: KUBE-SVC-xxx  tcp -- 0.0.0.0/0  0.0.0.0/0  tcp dpt:31234

# Follow the chain
sudo iptables -t nat -L KUBE-SVC-xxxx -n
# Should show DNAT rules pointing to pod IPs
```

## Step 7: Test with Source IP Considerations

If your service has `externalTrafficPolicy: Local`, traffic is only forwarded if the destination node has a local pod endpoint:

```bash
# Check the externalTrafficPolicy setting
kubectl get svc my-nodeport-service -o jsonpath='{.spec.externalTrafficPolicy}'
# If "Local", traffic to nodes without a local pod will fail

# Change to Cluster if needed (accepts traffic on all nodes)
kubectl patch svc my-nodeport-service -p '{"spec": {"externalTrafficPolicy": "Cluster"}}'
```

## Checking Which Nodes Have Pods

```bash
# For externalTrafficPolicy: Local, only connect to nodes with local pods
kubectl get pods -o wide | grep my-app | awk '{print $7}'
# Connect to those nodes' IPs on the NodePort
```

## Security Group / Cloud Provider

If running on a cloud provider (even bare-metal VMs may have security groups):

```bash
# Verify the NodePort range (30000-32767) is open in your cloud security group
# AWS: EC2 Security Groups → Inbound Rules
# GCP: Firewall Rules → Allow TCP 30000-32767
# Azure: NSG → Inbound Security Rules
```

NodePort issues are almost always caused by firewall rules blocking the port or `externalTrafficPolicy: Local` when pods don't run on all nodes.
