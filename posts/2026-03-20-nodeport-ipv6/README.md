# How to Configure IPv6 NodePort Services in Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Kubernetes, IPv6, NodePort, Services, Dual-Stack, External Access

Description: Configure Kubernetes NodePort Services for IPv6, understand how kube-proxy opens NodePorts on IPv6 node addresses, and test external IPv6 access to Kubernetes services via NodePorts.

## Introduction

NodePort Services expose Kubernetes services on a port across all cluster nodes. In dual-stack clusters, kube-proxy opens NodePorts on both IPv4 and IPv6 node addresses. This means external clients can reach NodePort services using either the node's IPv4 or IPv6 address. For a NodePort service to be accessible via IPv6, nodes must have IPv6 addresses and the correct firewall rules must allow the NodePort range (default: 30000-32767).

## Create NodePort Service with Dual-Stack

```yaml
# nodeport-dual-stack.yaml

apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  selector:
    app: web
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080  # Optional: specify port
  ipFamilyPolicy: PreferDualStack
  ipFamilies: [IPv4, IPv6]
  type: NodePort
```

```bash
kubectl apply -f nodeport-dual-stack.yaml

# Verify NodePort
kubectl get svc web-nodeport
# NAME           TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# web-nodeport   NodePort   10.96.x.x      <none>        80:30080/TCP   10s

# Check both ClusterIPs
kubectl get svc web-nodeport -o jsonpath='{.spec.clusterIPs}'
# ["10.96.x.x","fd00:10:96::x"]

# Check nodePort assigned
kubectl get svc web-nodeport -o jsonpath='{.spec.ports[0].nodePort}'
# 30080
```

## Access NodePort via IPv6

```bash
# Get node IPv6 address
NODE_IPV6=$(kubectl get node worker1 \
    -o jsonpath='{range .status.addresses[?(@.type=="InternalIP")]}{.address}{"\n"}{end}' | \
    grep ":")

echo "Node IPv6: $NODE_IPV6"

# Access service via node's IPv6 address and NodePort
curl -6 "http://[$NODE_IPV6]:30080/"

# Test from external client
# Ensure firewall allows port 30080 from external IPv6 clients
curl -6 "http://[2001:db8:node::1]:30080/"

# Access all nodes
for NODE in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
    NODE_IPV6=$(kubectl get node "$NODE" \
        -o jsonpath='{range .status.addresses[*]}{.address}{"\n"}{end}' | grep ":")
    echo "Node: $NODE, IPv6: $NODE_IPV6"
    curl -6 -s --connect-timeout 3 "http://[$NODE_IPV6]:30080/" && \
        echo "  -> ACCESSIBLE" || echo "  -> NOT ACCESSIBLE"
done
```

## Firewall Rules for NodePort IPv6 Access

```bash
# On each Kubernetes worker node, allow NodePort range via IPv6

# Using ip6tables
sudo ip6tables -A INPUT -p tcp \
    --dport 30000:32767 \
    -j ACCEPT

# For specific sources only
sudo ip6tables -A INPUT -p tcp \
    -s 2001:db8:clients::/48 \
    --dport 30000:32767 \
    -j ACCEPT

# In cloud environments (GCP):
gcloud compute firewall-rules create allow-nodeport-ipv6 \
    --network=vpc-main \
    --direction=INGRESS \
    --source-ranges="::/0" \
    --rules=tcp:30000-32767 \
    --target-tags=kubernetes-node

# In cloud environments (AWS):
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxx \
    --ip-permissions '[{"IpProtocol":"tcp","FromPort":30000,"ToPort":32767,"Ipv6Ranges":[{"CidrIpv6":"::/0"}]}]'
```

## kube-proxy and IPv6 NodePort Rules

```bash
# Verify kube-proxy created ip6tables rules for NodePort
sudo ip6tables -t nat -L KUBE-NODEPORTS -n

# Should show:
# KUBE-SVC-xxx  tcp  --  ::/0  ::/0  /* web-nodeport */ tcp dpt:30080

# For IPVS mode:
sudo ipvsadm -Ln | grep -B2 "30080" | grep "::"

# Check kube-proxy is managing IPv6 NodePorts
kubectl -n kube-system logs daemonset/kube-proxy | \
    grep -i "NodePort\|30080"
```

## IPv6-Only NodePort Service

```yaml
# IPv6-only NodePort (accessible only via IPv6 node addresses)
apiVersion: v1
kind: Service
metadata:
  name: api-nodeport-v6
spec:
  selector:
    app: api
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30088
  ipFamilyPolicy: SingleStack
  ipFamilies: [IPv6]
  type: NodePort
```

## Conclusion

Kubernetes NodePort Services in dual-stack clusters open the NodePort on both IPv4 and IPv6 node addresses through kube-proxy's ip6tables (or IPVS) rules. Set `ipFamilyPolicy: PreferDualStack` to get both IPv4 and IPv6 ClusterIPs for the service. Access via IPv6 using `http://[node-ipv6]:nodeport/`. Ensure firewall rules allow the NodePort range (30000-32767) for IPv6 clients on each node. Verify with `ip6tables -t nat -L KUBE-NODEPORTS -n` to see kube-proxy's IPv6 NodePort rules.
