# How to Use Litmus Chaos with Dapr on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Litmus Chaos, Kubernetes, Chaos Engineering, Resilience

Description: Use Litmus Chaos to run structured chaos experiments against Dapr services on Kubernetes, including pod deletion, network loss, and container kill scenarios.

---

## What is Litmus Chaos?

Litmus is a CNCF chaos engineering framework that provides a library of pre-built chaos experiments (ChaosHub) and a Kubernetes-native workflow engine. For Dapr applications, Litmus experiments can target app pods, Dapr sidecars, and dependent infrastructure like Redis or Kafka.

## Install Litmus on Kubernetes

```bash
# Install Litmus using Helm
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update

kubectl create namespace litmus

helm install chaos litmuschaos/litmus \
  --namespace litmus \
  --set portal.frontend.service.type=NodePort

# Verify installation
kubectl get pods -n litmus
```

## Create a Service Account for Experiments

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: litmus-admin
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: litmus-admin-role
rules:
  - apiGroups: ["", "apps", "litmuschaos.io"]
    resources: ["pods", "deployments", "chaosengines", "chaosexperiments", "chaosresults"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: litmus-admin-binding
subjects:
  - kind: ServiceAccount
    name: litmus-admin
    namespace: default
roleRef:
  kind: ClusterRole
  name: litmus-admin-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f litmus-rbac.yaml
```

## Install a Dapr Pod-Delete Experiment

```bash
# Install the pod-delete experiment from ChaosHub
kubectl apply -f https://hub.litmuschaos.io/api/chaos/3.0.0?file=charts/generic/pod-delete/experiment.yaml -n default
```

## Run a ChaosEngine Against a Dapr Service

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: dapr-pod-chaos
  namespace: default
spec:
  appinfo:
    appns: default
    applabel: "app=orderservice"
    appkind: deployment
  annotationCheck: "false"
  engineState: "active"
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CHAOS_INTERVAL
              value: "15"
            - name: FORCE
              value: "false"
            - name: PODS_AFFECTED_PERC
              value: "50"
```

```bash
kubectl apply -f dapr-pod-chaos.yaml

# Monitor chaos result
kubectl describe chaosresult dapr-pod-chaos-pod-delete -n default
```

## Verify Dapr Resiliency During Experiment

Watch Dapr logs and metrics during the chaos run:

```bash
# Terminal 1: watch sidecar logs for retries
kubectl logs -l app=callerservice -c daprd -f | grep -E "retry|error|circuit"

# Terminal 2: check chaos result status
watch kubectl get chaosresult -n default
```

## Interpret Chaos Results

```bash
kubectl get chaosresult dapr-pod-chaos-pod-delete -n default -o yaml
```

Look for `verdict: Pass` - this means Dapr's resiliency policy maintained service availability despite pod deletions. A `verdict: Fail` indicates your retry or circuit breaker configuration needs tuning.

## Summary

Litmus Chaos provides a structured, Kubernetes-native way to run chaos experiments against Dapr services using ChaosHub experiments and the ChaosEngine CRD. By combining pod-delete and network-loss experiments with Dapr resiliency policies, you can build confidence that your microservices recover automatically from infrastructure failures.
