# How to Capture IPv4 Packets on a Kubernetes Pod Network

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, tcpdump, Pod Network, Containers, Packet Capture

Description: Learn how to capture network packets on Kubernetes pod networks using tcpdump inside pods, ephemeral debug containers, or directly on the node's pod network interface for troubleshooting container...

## Kubernetes Network Capture Challenges

Kubernetes pods have isolated network namespaces. To capture pod traffic you have several options:
1. Run tcpdump inside the pod (if it has the binary)
2. Use an ephemeral debug container (Kubernetes 1.23+)
3. Capture on the host node's pod network interface (veth pair)
4. Use a DaemonSet-based capture tool

## Step 1: Capture Inside a Pod

```bash
# Check if tcpdump is available in the pod

kubectl exec -it my-pod -- which tcpdump

# If available, capture directly
kubectl exec -it my-pod -- tcpdump -i eth0 -n -w /tmp/capture.pcap

# Copy capture to local machine
kubectl cp my-pod:/tmp/capture.pcap /tmp/pod-capture.pcap
wireshark /tmp/pod-capture.pcap

# Real-time pipe to local Wireshark
kubectl exec -it my-pod -- tcpdump -i eth0 -n -w - | wireshark -k -i -
```

## Step 2: Use Ephemeral Debug Container (Kubernetes 1.23+)

```bash
# Add a debug container with network tools to a running pod
# Uses the same network namespace as the target pod

kubectl debug -it my-pod \
    --image=nicolaka/netshoot \
    --target=my-container

# Inside the debug container:
tcpdump -i eth0 -n -w /tmp/capture.pcap 'not port 22'

# nicolaka/netshoot includes: tcpdump, tshark, wireshark, curl, nmap, iperf3
```

```bash
# One-liner to stream capture from debug container to local Wireshark
kubectl debug -it my-pod \
    --image=nicolaka/netshoot \
    --target=my-container \
    -- tcpdump -i eth0 -n -w - | wireshark -k -i -
```

## Step 3: Capture on Node via veth Interface

```bash
# Find which node the pod is running on
kubectl get pod my-pod -o wide
# Note the NODE column

# SSH to the node
ssh user@k8s-node-1

# Find the pod's network namespace
CONTAINER_ID=$(kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[0].containerID}' | sed 's/.*docker:\/\///')

# Find the veth pair
# Method 1: via nsenter
PID=$(docker inspect --format '{{.State.Pid}}' $CONTAINER_ID)
nsenter -t $PID -n ip link show eth0
# Note the interface index number

# Find matching veth on host
ip link | grep "if${INDEX}"
# Example output: veth1234abcd@if3
```

```bash
# Capture on the veth interface (captures all pod traffic)
VETH=$(ip link | grep "veth" | grep -v SLAVE | awk -F: '{print $2}' | head -1 | tr -d ' ')
sudo tcpdump -i $VETH -n -w /tmp/pod-veth-capture.pcap
```

## Step 4: Use ksniff Plugin

```bash
# Install ksniff (kubectl plugin for pod packet capture)
kubectl krew install sniff

# Capture from a pod (streams to Wireshark automatically)
kubectl sniff my-pod -n default

# Capture with filter
kubectl sniff my-pod -f "port 8080" -n default

# Capture to file instead of Wireshark
kubectl sniff my-pod -o /tmp/pod-capture.pcap -n default
```

## Step 5: Deploy DaemonSet-based Capture

```yaml
# tcpdump-daemonset.yaml
# Deploys tcpdump on every node for network debugging
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: packet-capture
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: packet-capture
  template:
    metadata:
      labels:
        app: packet-capture
    spec:
      hostNetwork: true    # Use host network namespace
      containers:
      - name: tcpdump
        image: nicolaka/netshoot
        command: ["tcpdump", "-i", "any", "-n",
                  "-w", "/captures/$(NODE_NAME).pcap",
                  "not port 22"]
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: captures
          mountPath: /captures
        securityContext:
          capabilities:
            add: [NET_ADMIN, NET_RAW]
      volumes:
      - name: captures
        hostPath:
          path: /tmp/k8s-captures
```

```bash
kubectl apply -f tcpdump-daemonset.yaml

# Get captures from nodes
kubectl get pods -n kube-system -l app=packet-capture

# Copy capture file
kubectl cp kube-system/packet-capture-xxxxx:/captures/node-name.pcap /tmp/
```

## Step 6: Capture with nsenter on Node

```bash
# SSH to the node where the pod runs
ssh user@k8s-worker-1

# Find pod's PID
CONTAINER=$(crictl ps | grep my-pod | awk '{print $1}')
PID=$(crictl inspect $CONTAINER | jq '.info.pid')

# Capture in pod's network namespace using nsenter
sudo nsenter -t $PID -n tcpdump -i eth0 -n -w /tmp/pod-capture.pcap

# Or pipe to local Wireshark via SSH
ssh user@k8s-worker-1 "sudo nsenter -t $PID -n tcpdump -i eth0 -n -w -" | \
    wireshark -k -i -
```

## Step 7: Capture CNI-Specific Traffic

```bash
# Calico - capture BGP and overlay traffic
# On node
sudo tcpdump -i tunl0 -n -w /tmp/calico-tunnel.pcap

# Flannel - capture VXLAN overlay
sudo tcpdump -i flannel.1 -n -w /tmp/flannel-overlay.pcap

# Capture encapsulated traffic and inner packets
# Inner packets are visible after decapsulation in Wireshark
# Wireshark filter: vxlan or geneve
```

## Conclusion

Kubernetes pod network capture has multiple approaches: run `tcpdump` directly inside pods, use `kubectl debug` with `nicolaka/netshoot` image for ephemeral debug containers, or install the `kubectl sniff` plugin for automatic Wireshark streaming. On the node level, identify the pod's veth pair and capture there, or use `nsenter -t [PID] -n` to enter the pod's network namespace. For cluster-wide capture, deploy a DaemonSet with `hostNetwork: true` and `NET_RAW` capabilities.
