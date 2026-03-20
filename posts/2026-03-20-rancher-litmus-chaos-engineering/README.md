# How to Set Up Chaos Engineering with Litmus on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, litmus, chaos-engineering, resilience, kubernetes

Description: A guide to setting up LitmusChaos on Rancher-managed Kubernetes clusters for chaos engineering experiments to improve application resilience.

## Overview

Chaos engineering is the practice of deliberately introducing failures into systems to test their resilience and identify weaknesses before they cause incidents. LitmusChaos is a CNCF-graduated chaos engineering platform for Kubernetes. This guide covers installing LitmusChaos on Rancher-managed clusters, running chaos experiments, and integrating chaos into CI/CD pipelines.

## What Is LitmusChaos?

LitmusChaos provides a catalog of chaos experiments (ChaosHub) including pod deletion, node drain, CPU stress, network packet loss, disk I/O saturation, and more. It uses Kubernetes CRDs (ChaosEngine, ChaosExperiment, ChaosResult) and provides a web UI (Litmus Portal) for managing experiments.

## Step 1: Install LitmusChaos

```bash
# Add LitmusChaos Helm repository
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update

# Install LitmusChaos (namespace-scoped)
kubectl create namespace litmus

helm install chaos litmuschaos/litmus \
  --namespace litmus \
  --set portal.frontend.service.type=ClusterIP \
  --set portal.server.graphqlServer.replicaCount=1

# For Rancher with LoadBalancer
kubectl patch svc litmusportal-frontend-service -n litmus \
  -p '{"spec": {"type": "LoadBalancer"}}'

# Get the Litmus Portal URL
kubectl get svc litmusportal-frontend-service -n litmus
```

## Step 2: Access Litmus Portal

```bash
# Default credentials
# Username: admin
# Password: litmus

# Change password immediately after first login
```

## Step 3: Install Chaos Experiments from ChaosHub

```bash
# Install common chaos experiments
kubectl apply -f https://hub.litmuschaos.io/api/chaos/master?file=charts/generic/experiments.yaml -n litmus

# Or install specific experiments
kubectl apply -f https://hub.litmuschaos.io/api/chaos/master?file=charts/generic/pod-delete/experiments.yaml
kubectl apply -f https://hub.litmuschaos.io/api/chaos/master?file=charts/generic/pod-cpu-hog/experiments.yaml
kubectl apply -f https://hub.litmuschaos.io/api/chaos/master?file=charts/generic/node-drain/experiments.yaml
```

## Step 4: Create a ChaosEngine

A ChaosEngine defines the target application and the experiments to run:

### Pod Delete Experiment

```yaml
# Test application resilience to pod deletion
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: webapp-pod-delete-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=webapp"
    appkind: deployment

  # Monitor experiment using Prometheus
  monitoring: true
  jobCleanUpPolicy: retain

  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"     # Run for 60 seconds
            - name: CHAOS_INTERVAL
              value: "10"     # Delete a pod every 10 seconds
            - name: FORCE
              value: "false"  # Graceful deletion
            - name: PODS_AFFECTED_PERC
              value: "50"    # Delete 50% of matching pods
```

### CPU Stress Experiment

```yaml
# Stress CPU on a pod to test resource limits and HPA
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: api-cpu-stress
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=api-service"
    appkind: deployment

  experiments:
    - name: pod-cpu-hog
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "120"   # 2 minutes
            - name: CPU_CORES
              value: "2"     # Hog 2 CPU cores
            - name: PODS_AFFECTED_PERC
              value: "100"   # Affect all pods
```

### Node Drain Experiment

```yaml
# Test cluster behavior when a node is drained
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: node-drain-test
  namespace: litmus
spec:
  engineState: active
  chaosServiceAccount: litmus-admin

  experiments:
    - name: node-drain
      spec:
        components:
          env:
            - name: TARGET_NODE
              value: "worker-node-02"   # Target specific node
            - name: TOTAL_CHAOS_DURATION
              value: "60"
```

## Step 5: Monitor Chaos Experiments

```yaml
# PrometheusRule to track chaos experiment outcomes
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: chaos-monitoring
  namespace: cattle-monitoring-system
spec:
  groups:
    - name: chaos-experiments
      rules:
        - alert: ChaosExperimentFailed
          expr: litmuschaos_experiment_verdict{verdict="Fail"} > 0
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: "Chaos experiment failed: {{ $labels.chaosengine }}"
```

## Step 6: Integrate Chaos into CI/CD

```yaml
# GitHub Actions: Run chaos experiments on staging
name: Chaos Engineering Tests
on:
  schedule:
    - cron: '0 2 * * *'   # Nightly chaos tests

jobs:
  chaos-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        run: |
          echo "${{ secrets.STAGING_KUBECONFIG }}" | base64 -d > kubeconfig.yaml
          export KUBECONFIG=kubeconfig.yaml

      - name: Run pod delete chaos
        run: |
          kubectl apply -f chaos/pod-delete-engine.yaml
          kubectl wait --for=jsonpath='{.status.engineStatus}'=completed \
            chaosengine/webapp-pod-delete-chaos \
            -n production \
            --timeout=300s

      - name: Check chaos result
        run: |
          VERDICT=$(kubectl get chaosresult \
            webapp-pod-delete-chaos-pod-delete \
            -n production \
            -o jsonpath='{.status.experimentstatus.verdict}')

          if [ "${VERDICT}" != "Pass" ]; then
            echo "FAIL: Chaos test failed with verdict: ${VERDICT}"
            exit 1
          fi
          echo "PASS: Application survived pod deletion chaos"

      - name: Cleanup
        if: always()
        run: kubectl delete chaosengine webapp-pod-delete-chaos -n production
```

## Step 7: Define SLOs for Chaos Tests

```yaml
# ProbeConfiguration: Define success criteria for chaos experiments
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: webapp-with-probes
  namespace: production
spec:
  experiments:
    - name: pod-delete
      spec:
        probe:
          # HTTP probe: Check API returns 200 during chaos
          - name: api-health-check
            type: httpProbe
            httpProbe/inputs:
              url: "http://webapp.production.svc/health"
              insecureSkipVerify: false
              responseTimeout: 5000
              method:
                get:
                  criteria: "=="
                  responseCode: "200"
            mode: Continuous
            runProperties:
              probeTimeout: 5s
              interval: 5s
              retry: 3

          # Prometheus probe: Verify error rate stays below 1%
          - name: error-rate-probe
            type: promProbe
            promProbe/inputs:
              endpoint: http://prometheus.cattle-monitoring-system.svc:9090
              query: |
                sum(rate(http_requests_total{status=~"5.."}[1m])) /
                sum(rate(http_requests_total[1m])) < 0.01
              comparator:
                type: float
                criteria: "=="
                value: "1"    # Query must return 1 (true)
            mode: Edge
```

## Conclusion

LitmusChaos on Rancher provides a powerful platform for testing application resilience through controlled chaos experiments. Starting with simple pod deletion tests and progressing to node drain and network partition experiments helps uncover hidden failure modes. Integrating chaos experiments into your CI/CD pipeline with automated pass/fail verdicts ensures that resilience is maintained as applications evolve. Always start chaos experiments in staging environments and work up to production with careful monitoring and clear rollback procedures.
