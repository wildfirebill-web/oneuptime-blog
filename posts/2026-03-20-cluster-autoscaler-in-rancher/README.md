# How to Configure Cluster Autoscaler in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Cluster Autoscaler, Kubernetes, Auto Scaling, AWS, Cost Optimization

Description: Configure the Kubernetes Cluster Autoscaler in Rancher to automatically add and remove worker nodes based on pending pod requests and underutilization.

## Introduction

The Cluster Autoscaler (CA) automatically adjusts the number of nodes in a cluster based on pod scheduling needs. When pods are pending due to insufficient resources, CA adds nodes. When nodes are underutilized, CA removes them. This reduces cloud costs while ensuring applications always have capacity.

## Prerequisites

- Rancher cluster running on a cloud provider (AWS, GCP, Azure)
- Node groups/ASGs configured on the cloud provider
- IAM permissions for the autoscaler to manage node groups

## Step 1: Configure AWS IAM Policy

The Cluster Autoscaler needs permissions to describe and modify Auto Scaling Groups:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": "*"
    }
  ]
}
```

## Step 2: Tag Your Auto Scaling Groups

```bash
# Tag ASGs so the Cluster Autoscaler can discover them

aws autoscaling create-or-update-tags \
  --tags \
    ResourceId=my-worker-asg \
    ResourceType=auto-scaling-group \
    Key=k8s.io/cluster-autoscaler/enabled \
    Value=true \
    PropagateAtLaunch=false \
    ResourceId=my-worker-asg \
    ResourceType=auto-scaling-group \
    Key=k8s.io/cluster-autoscaler/production \
    Value=owned \
    PropagateAtLaunch=false
```

## Step 3: Deploy the Cluster Autoscaler

```yaml
# cluster-autoscaler-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.2
          name: cluster-autoscaler
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste       # Pick the node group that wastes the least resources
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled=true,k8s.io/cluster-autoscaler/production=owned
            - --balance-similar-node-groups  # Keep node groups balanced
            - --scale-down-delay-after-add=10m
            - --scale-down-unneeded-time=10m
          resources:
            requests:
              cpu: 100m
              memory: 300Mi
```

## Step 4: Configure RBAC

```yaml
# cluster-autoscaler-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "update"]
```

## Step 5: Verify Autoscaler Activity

```bash
# Watch autoscaler logs
kubectl logs -n kube-system deployment/cluster-autoscaler -f

# Check scale-up events
kubectl get events -n kube-system \
  --field-selector reason=TriggeredScaleUp

# View current node group sizes
kubectl get nodes
```

## Conclusion

The Cluster Autoscaler on Rancher enables cost-effective elastic infrastructure. Combine it with KEDA for application-level scaling and Pod Disruption Budgets to ensure graceful scale-down during node termination. Set appropriate `--scale-down-delay` values to prevent thrashing during variable load periods.
