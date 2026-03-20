# How to Monitor Elemental Machine State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Elemental, Monitoring, Kubernetes, Edge, Observability

Description: Monitor the health and state of Elemental-managed nodes using Rancher's machine inventory, conditions, and metrics.

## Introduction

Monitoring the state of Elemental machines is essential for maintaining a healthy edge or bare metal fleet. The Elemental Operator exposes machine state through Kubernetes resource conditions, which can be queried directly, integrated with monitoring systems, or used to trigger automated remediation.

## Understanding Machine States

Elemental machines can be in these states:

| State | Description |
|-------|-------------|
| Pending | Registered but not yet adopted by a cluster |
| Adopted | Member of a Kubernetes cluster |
| Ready | Healthy and accepting workloads |
| NotReady | Unhealthy or unreachable |
| Resetting | Undergoing a reset operation |
| Upgrading | OS upgrade in progress |

## Querying Machine State

```bash
# Get all machines with their current state
kubectl get machineinventory -n fleet-default \
  -o custom-columns=\
'NAME:.metadata.name,\
ADOPTED:.spec.machineRef.name,\
AGE:.metadata.creationTimestamp'

# Get machines in error state
kubectl get machineinventory -n fleet-default \
  -o json | jq '.items[] | select(.status.conditions[] | .type == "Ready" and .status == "False") | {name: .metadata.name, reason: .status.conditions[] | select(.type == "Ready") | .reason}'

# Watch for state changes
kubectl get machineinventory -n fleet-default --watch
```

## Checking Machine Conditions

```bash
# Get conditions for all machines
kubectl get machineinventory -n fleet-default \
  -o json | jq '.items[] | {
    name: .metadata.name,
    conditions: .status.conditions | map({type, status, reason, message})
  }'

# Check a specific machine's conditions
kubectl get machineinventory -n fleet-default m-abc12345 \
  -o jsonpath='{.status.conditions}' | jq .
```

## Setting Up Prometheus Monitoring

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: elemental-machine-alerts
  namespace: monitoring
spec:
  groups:
    - name: elemental.machines
      interval: 60s
      rules:
        # Alert when machines go offline
        - alert: ElementalMachineNotReady
          expr: |
            kube_customresource_elemental_machine_inventory_status_condition{
              condition="Ready",
              status="False"
            } == 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Elemental machine {{ $labels.name }} is not ready"
            description: "Machine has been in NotReady state for more than 5 minutes"
```

## Monitoring with kubectl Plugins

```bash
# Install krew for kubectl plugins
kubectl krew install resource-capacity

# View resource usage across all machines
kubectl resource-capacity --pods --sort cpu.limit

# Create a simple dashboard script
cat > /usr/local/bin/elemental-status << 'EOF'
#!/bin/bash
echo "=== Elemental Fleet Status ==="
echo ""
echo "Total machines:"
kubectl get machineinventory -n fleet-default --no-headers | wc -l

echo ""
echo "Adopted machines:"
kubectl get machineinventory -n fleet-default \
  -o json | jq '[.items[] | select(.spec.machineRef != null)] | length'

echo ""
echo "Available (not adopted):"
kubectl get machineinventory -n fleet-default \
  -o json | jq '[.items[] | select(.spec.machineRef == null)] | length'

echo ""
echo "Machines by location:"
kubectl get machineinventory -n fleet-default \
  -o json | jq '[.items[].metadata.labels.location] | group_by(.) | map({location: .[0], count: length})'
EOF
chmod +x /usr/local/bin/elemental-status
```

## Grafana Dashboard Integration

```bash
# Export machine inventory metrics for Grafana
# Create a custom metrics exporter script
cat > /usr/local/bin/elemental-metrics-exporter << 'EOF'
#!/bin/bash
# Export Elemental metrics in Prometheus format

NAMESPACE=fleet-default
TOTAL=$(kubectl get machineinventory -n $NAMESPACE --no-headers | wc -l)
ADOPTED=$(kubectl get machineinventory -n $NAMESPACE -o json | jq '[.items[] | select(.spec.machineRef != null)] | length')
AVAILABLE=$((TOTAL - ADOPTED))

echo "# HELP elemental_machines_total Total number of registered Elemental machines"
echo "# TYPE elemental_machines_total gauge"
echo "elemental_machines_total ${TOTAL}"
echo ""
echo "# HELP elemental_machines_adopted Number of machines adopted by a cluster"
echo "# TYPE elemental_machines_adopted gauge"
echo "elemental_machines_adopted ${ADOPTED}"
echo ""
echo "# HELP elemental_machines_available Number of available (not adopted) machines"
echo "# TYPE elemental_machines_available gauge"
echo "elemental_machines_available ${AVAILABLE}"
EOF
chmod +x /usr/local/bin/elemental-metrics-exporter
```

## Conclusion

Monitoring Elemental machine state through Kubernetes conditions, custom scripts, and Prometheus integration gives you comprehensive visibility into your edge fleet. By setting up alerts for machines that fail to register or become unhealthy, you can proactively address issues before they impact workloads. The Kubernetes-native approach means your existing monitoring stack can be extended to cover your entire bare metal and edge fleet.
