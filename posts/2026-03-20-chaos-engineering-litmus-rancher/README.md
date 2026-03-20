# How to Set Up Chaos Engineering with Litmus on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Chaos Engineering, Litmus, Resilience, Kubernetes, Testing

Description: Set up chaos engineering with LitmusChaos on Rancher to test application and cluster resilience through pod kills, network failures, CPU stress, and node drains in controlled experiments.

## Introduction

Chaos engineering proactively tests how systems behave under failure conditions. Rather than discovering failures during incidents, chaos engineering deliberately introduces controlled failures to identify weaknesses before they impact users. LitmusChaos is a CNCF project that provides Kubernetes-native chaos experiments, integrating naturally with Rancher-managed clusters.

## Step 1: Install LitmusChaos

```bash
# Install via Helm

helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update

helm install litmus litmuschaos/litmus \
  --namespace litmus \
  --create-namespace \
  --set portal.frontend.service.type=ClusterIP

# Access Litmus Portal via port-forward
kubectl port-forward svc/litmus-frontend-service 9091:9091 -n litmus
```

## Step 2: Install Chaos Experiments

```bash
# Install standard chaos experiment hub
kubectl apply -f https://hub.litmuschaos.io/api/chaos/2.14.0?file=charts/generic/experiments.yaml -n litmus

# Verify experiments are available
kubectl get chaosexperiments -n litmus | head -20
```

## Step 3: Pod Delete Experiment

```yaml
# Test that application recovers from pod failures
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-delete-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=api-server"
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        probe:
          # Verify application stays available during chaos
          - name: api-availability-probe
            type: httpProbe
            mode: Continuous
            runProperties:
              probeTimeout: 5
              interval: 2
              attempt: 10
            httpProbe/inputs:
              url: "http://api-server.production.svc/health"
              insecureSkipVerify: true
              responseCode: "200"
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"        # 60 seconds of chaos
            - name: CHAOS_INTERVAL
              value: "10"        # Delete pod every 10 seconds
            - name: FORCE
              value: "false"     # Graceful termination
```

## Step 4: Network Chaos Experiments

```yaml
# Pod network latency - inject 200ms latency
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: network-latency-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=frontend"
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-network-latency
      spec:
        probe:
          - name: latency-tolerance-probe
            type: cmdProbe
            mode: Edge
            cmdProbe/inputs:
              command: "kubectl get deployment frontend -n production -o jsonpath='{.status.availableReplicas}'"
              source: ""
              comparator:
                type: int
                criteria: ">="
                value: "2"
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "120"
            - name: NETWORK_LATENCY
              value: "200"     # 200ms latency
            - name: JITTER
              value: "20"      # ±20ms jitter
            - name: DESTINATION_HOSTS
              value: "backend-service.production.svc"
```

## Step 5: Node Drain Experiment

```yaml
# Test cluster behavior when a node goes offline
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: node-drain-chaos
  namespace: litmus
spec:
  appinfo:
    appns: production
    applabel: "app=critical-service"
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: node-drain
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "120"
            - name: NODE_LABEL
              value: "type=worker"    # Drain worker nodes
            - name: REVERT_CHAOS
              value: "true"           # Re-enable node after chaos
```

## Step 6: CPU and Memory Stress

```yaml
# Simulate resource contention
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: cpu-stress-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=database"
    appkind: statefulset
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-cpu-hog
      spec:
        probe:
          - name: db-response-time-probe
            type: httpProbe
            mode: Continuous
            httpProbe/inputs:
              url: "http://database-svc.production.svc:5432/health"
              responseCode: "200"
            runProperties:
              probeTimeout: 3
              interval: 1
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "120"
            - name: CPU_CORES
              value: "2"       # Stress 2 CPU cores
            - name: CPU_LOAD
              value: "80"      # 80% CPU load
```

## Step 7: Chaos Workflow Integration

```yaml
# Run chaos experiments in CI/CD pipeline
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: chaos-test-suite
  namespace: litmus
spec:
  entrypoint: chaos-suite
  templates:
    - name: chaos-suite
      steps:
        - - name: pod-delete
            template: run-chaos
            arguments:
              parameters:
                - name: experiment
                  value: pod-delete-chaos.yaml
        - - name: network-latency
            template: run-chaos
            arguments:
              parameters:
                - name: experiment
                  value: network-latency-chaos.yaml
    - name: run-chaos
      inputs:
        parameters:
          - name: experiment
      container:
        image: litmuschaos/litmus-checker:latest
        command: [kubectl, apply, -f]
        args: ["{{inputs.parameters.experiment}}"]
```

## Conclusion

Chaos engineering with LitmusChaos on Rancher transforms disaster preparedness from reactive to proactive. Pod deletion, network latency injection, node drain, and resource stress experiments reveal weaknesses in application resilience, PodDisruptionBudgets, anti-affinity rules, and timeout configurations. Run chaos experiments regularly in staging environments, and for mature applications, introduce controlled chaos in production during low-traffic periods. The insights gained prevent real incidents and build team confidence in system resilience.
