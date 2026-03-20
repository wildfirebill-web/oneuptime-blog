# How to Perform Rancher HA Failover Testing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, High Availability, Failover Testing, Chaos Engineering, Resilience

Description: Test Rancher HA resilience by simulating node failures, network partitions, and etcd failures to validate your recovery time objectives.

## Introduction

HA configurations are only valuable if they actually work during failures. Regularly testing failover scenarios ensures your Rancher deployment meets its recovery time objectives (RTO) and that you understand the actual behavior during different failure types.

## Failover Test Plan

Document your tests and expected outcomes before running them:

| Test Scenario | Expected Behavior | Acceptable RTO |
|---|---|---|
| Single Rancher pod restart | Automatic recovery | < 30s |
| Single server node failure | Traffic rerouted | < 60s |
| etcd leader election | Automatic new leader | < 30s |
| Load balancer failure | VIP failover | < 10s |
| Network partition | Quorum maintained | < 120s |

## Test 1: Single Pod Failure

```bash
# Kill one Rancher pod and verify recovery
POD=$(kubectl get pods -n cattle-system -l app=rancher -o name | head -1)
kubectl delete $POD -n cattle-system

# Immediately test that Rancher is still accessible
while true; do
    STATUS=$(curl -k -s -o /dev/null -w "%{http_code}" https://rancher.example.com/healthz)
    TIMESTAMP=$(date +%T)
    echo "$TIMESTAMP: HTTP $STATUS"
    sleep 2
done
# Should remain 200; brief blip is acceptable
```

## Test 2: Single Node Failure

```bash
# Simulate node failure by stopping RKE2/K3s on one node
ssh 10.0.0.11 "sudo systemctl stop rke2-server"

# Monitor Rancher availability from a separate terminal
while true; do
    STATUS=$(curl -k -s -o /dev/null -w "%{http_code}" https://rancher.example.com/healthz)
    echo "$(date +%T): HTTP $STATUS"
    sleep 5
done

# Check that other nodes maintain quorum
kubectl get nodes

# After testing, restore the node
ssh 10.0.0.11 "sudo systemctl start rke2-server"
```

## Test 3: etcd Leader Election Failure

```bash
# Find the current etcd leader
kubectl exec -n kube-system etcd-server-1 -- \
  etcdctl endpoint status --write-out=table \
  --endpoints=https://10.0.0.11:2379,https://10.0.0.12:2379,https://10.0.0.13:2379

# Kill the etcd leader's node
LEADER_NODE="10.0.0.11"  # Replace with actual leader IP
ssh $LEADER_NODE "sudo systemctl stop rke2-server"

# Measure time until new leader is elected
START=$(date +%s)
while ! kubectl get nodes &>/dev/null; do sleep 1; done
END=$(date +%s)
echo "Leader election took: $((END-START)) seconds"
```

## Test 4: Network Partition Simulation

```bash
# Block communication between two nodes using iptables
ssh 10.0.0.11 "sudo iptables -A INPUT -s 10.0.0.12 -j DROP"
ssh 10.0.0.11 "sudo iptables -A OUTPUT -d 10.0.0.12 -j DROP"

# Monitor cluster health during partition
kubectl get nodes -w

# Restore network
ssh 10.0.0.11 "sudo iptables -F"
```

## Test 5: Load Balancer Failover

```bash
# Stop the primary HAProxy/NGINX node
# The VIP should move to the secondary within Keepalived's advertisement interval

# Check VIP location before
ip addr show | grep "10.0.0.10"

# Stop the primary load balancer
sudo systemctl stop keepalived

# Verify VIP has moved to secondary
# From another host:
curl -k https://10.0.0.10/healthz    # Should still respond
```

## Recording Results

```bash
# Save failover test results
cat > failover-test-$(date +%Y%m%d).md << 'EOF'
# Failover Test Results

Date: $(date)
Tester: your-name

| Test | Start Time | Recovery Time | Result |
|------|-----------|---------------|--------|
| Pod failure | 14:00:00 | 14:00:28 | PASS (28s) |
| Node failure | 14:10:00 | 14:11:05 | PASS (65s) |
EOF
```

## Conclusion

Regular failover testing is essential for maintaining confidence in your Rancher HA deployment. Run these tests quarterly and after any significant infrastructure changes. Document results to track your actual RTO over time and identify drift from your original HA design assumptions.
