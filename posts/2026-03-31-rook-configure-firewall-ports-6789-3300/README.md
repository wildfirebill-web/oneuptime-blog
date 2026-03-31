# How to Configure Firewall Ports 6789 and 3300 for Ceph Monitors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Monitor, Networking, Firewall

Description: Configure firewall rules for Ceph monitor ports 6789 (v1 msgr) and 3300 (v2 msgr2) to ensure secure monitor connectivity in Rook clusters.

---

## Ceph Monitor Port Requirements

Ceph monitors listen on two ports depending on the protocol version:

- **Port 6789**: The legacy `msgr1` (v1) protocol. Used by older clients and Ceph versions before Nautilus.
- **Port 3300**: The newer `msgr2` protocol. Introduced in Nautilus, supports encryption and authentication improvements. All Rook deployments use msgr2 by default.

Both ports must be accessible between all monitors, all OSDs, all MDS daemons, and all clients. In Kubernetes, Rook creates Services for each MON, so within-cluster traffic routes through Kubernetes networking. External clients connecting to Ceph still need these ports open at the network boundary.

## Checking Monitor Listening Ports

Verify which ports monitors are actually listening on:

```bash
kubectl -n rook-ceph exec -it rook-ceph-mon-a-<suffix> -- \
  ss -tlnp | grep -E "6789|3300"
```

Check the Kubernetes Services exposing monitor ports:

```bash
kubectl -n rook-ceph get svc -l app=rook-ceph-mon
```

## Configuring iptables Rules

On each Kubernetes node, allow Ceph monitor traffic:

```bash
# Allow msgr2 (v2 protocol)
sudo iptables -A INPUT -p tcp --dport 3300 -j ACCEPT

# Allow msgr1 (v1 legacy protocol)
sudo iptables -A INPUT -p tcp --dport 6789 -j ACCEPT

# Save rules
sudo iptables-save > /etc/iptables/rules.v4
```

## Configuring firewalld Rules

On RHEL/CentOS nodes using firewalld:

```bash
# Permanent rules for Ceph monitor ports
sudo firewall-cmd --permanent --add-port=3300/tcp
sudo firewall-cmd --permanent --add-port=6789/tcp

# Apply changes
sudo firewall-cmd --reload

# Verify
sudo firewall-cmd --list-ports
```

## Kubernetes NetworkPolicy for Monitor Access

If your cluster uses Kubernetes NetworkPolicies, create one that allows monitor traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ceph-mon
  namespace: rook-ceph
spec:
  podSelector:
    matchLabels:
      app: rook-ceph-mon
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 3300
    - protocol: TCP
      port: 6789
```

## Verifying Connectivity

Test port connectivity from an OSD pod to a monitor:

```bash
kubectl -n rook-ceph exec -it rook-ceph-osd-0-<suffix> -- \
  bash -c "timeout 3 bash -c '</dev/tcp/rook-ceph-mon-a/3300' && echo 'Port 3300 OK'"

kubectl -n rook-ceph exec -it rook-ceph-osd-0-<suffix> -- \
  bash -c "timeout 3 bash -c '</dev/tcp/rook-ceph-mon-a/6789' && echo 'Port 6789 OK'"
```

## Disabling Legacy msgr1 Protocol

For security hardening, disable the legacy v1 protocol entirely:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mon ms_bind_msgr1 false
```

After disabling v1, port 6789 is no longer needed.

## Summary

Ceph monitors require port 3300 (msgr2) and optionally 6789 (msgr1) open between all cluster daemons. Configure these via `iptables`, `firewalld`, or Kubernetes NetworkPolicies depending on your infrastructure. For new deployments, disabling msgr1 and operating exclusively on port 3300 simplifies firewall rules and improves security.
