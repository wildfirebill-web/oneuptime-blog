# How to Capture Packets from a Kubernetes Pod

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, Networking, Packet Capture, tcpdump, Wireshark, Debugging

Description: Learn how to capture network packets directly from a Kubernetes pod using tcpdump, kubectl debug, and ephemeral containers.

---

Capturing packets from a running Kubernetes pod is essential for debugging network issues, analyzing protocols, and verifying encryption. Several techniques work depending on your cluster access level.

---

## Method 1: kubectl exec into a Pod with tcpdump

```bash
# If the pod has tcpdump installed

kubectl exec -it my-pod -- tcpdump -i eth0 -w /tmp/capture.pcap

# Copy the capture file to your local machine
kubectl cp my-pod:/tmp/capture.pcap ./capture.pcap

# Open with Wireshark
wireshark capture.pcap
```

---

## Method 2: Ephemeral Debug Container (Kubernetes 1.23+)

```bash
# Inject a debug container that shares the target pod's network namespace
kubectl debug -it my-pod   --image=nicolaka/netshoot   --target=my-container   -- tcpdump -i eth0 -nn -w /tmp/capture.pcap

# In another terminal, copy the file
kubectl cp my-pod:/tmp/capture.pcap ./capture.pcap -c debugger
```

---

## Method 3: Capture on the Node

```bash
# SSH to the node running the pod
# Find the pod's network interface
kubectl get pod my-pod -o wide  # Get node name
ssh node01

# Find the veth interface for the pod
POD_IP=$(kubectl get pod my-pod -o jsonpath='{.status.podIP}')
VETH=$(ip route get ${POD_IP} | grep dev | awk '{print $3}')

# Capture on the veth interface
sudo tcpdump -i ${VETH} -w /tmp/pod-capture.pcap
```

---

## Method 4: ksniff Plugin

```bash
# Install ksniff kubectl plugin
kubectl krew install sniff

# Capture and open directly in Wireshark
kubectl sniff my-pod -n default -o capture.pcap
```

---

## Filter Useful Traffic

```bash
# Capture only HTTP/HTTPS
tcpdump -i eth0 'port 80 or port 443' -w capture.pcap

# Capture only DNS
tcpdump -i eth0 'port 53' -w dns-capture.pcap

# Capture between specific pods
tcpdump -i eth0 host 10.0.1.5 -w pod-to-pod.pcap
```

---

## Summary

Use `kubectl exec` for quick captures if `tcpdump` is in the image, or inject an ephemeral debug container using `kubectl debug` with `--target` to share the pod's network namespace. For longer captures, run `tcpdump` on the node's veth interface. Transfer the `.pcap` file with `kubectl cp` and analyze with Wireshark.
