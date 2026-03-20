# How to Configure AWS ECR with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, AWS, ECR, Container Registry

Description: Configure Amazon Elastic Container Registry (ECR) with Rancher clusters to securely pull and manage container images using AWS IAM authentication.

## Introduction

Amazon Elastic Container Registry (ECR) is AWS's managed container registry service. ECR authentication uses temporary tokens that expire every 12 hours, which requires a different approach than static registry credentials. This guide covers multiple methods for integrating ECR with Rancher clusters, from manual configuration to automated token refresh.

## Prerequisites

- Rancher cluster running on AWS (EKS, EC2, or on-premises with AWS access)
- AWS CLI configured with appropriate permissions
- IAM permissions to create ECR repositories and IAM roles
- kubectl access to your cluster

## Step 1: Create an ECR Repository

```bash
# Create an ECR repository

aws ecr create-repository \
  --repository-name my-app \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true

# Get the repository URI
aws ecr describe-repositories \
  --repository-names my-app \
  --query 'repositories[0].repositoryUri' \
  --output text
```

## Step 2: Authenticate and Get ECR Token

```bash
# Get ECR login token (valid for 12 hours)
aws ecr get-login-password --region us-east-1 | \
  docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create a Kubernetes secret from the ECR token
kubectl create secret docker-registry ecr-credentials \
  --docker-server=123456789012.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  --namespace=production
```

## Step 3: Automate ECR Token Refresh with a CronJob

Since ECR tokens expire every 12 hours, automate renewal with a CronJob:

```yaml
# ecr-token-refresh.yaml - Automated ECR token refresh
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ecr-token-refresher
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ecr-token-refresher
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["create", "get", "patch", "update", "delete"]
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["get", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ecr-token-refresher
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ecr-token-refresher
subjects:
  - kind: ServiceAccount
    name: ecr-token-refresher
    namespace: production
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ecr-token-refresh
  namespace: production
spec:
  # Run every 10 hours (token expires in 12h)
  schedule: "0 */10 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ecr-token-refresher
          restartPolicy: Never
          containers:
            - name: ecr-token-refresh
              image: amazon/aws-cli:latest
              env:
                - name: AWS_REGION
                  value: us-east-1
                - name: ECR_REGISTRY
                  value: 123456789012.dkr.ecr.us-east-1.amazonaws.com
                - name: SECRET_NAME
                  value: ecr-credentials
                - name: NAMESPACE
                  value: production
              command:
                - /bin/sh
                - -c
                - |
                  # Get new ECR token
                  TOKEN=$(aws ecr get-login-password --region $AWS_REGION)

                  # Create or update the secret
                  kubectl create secret docker-registry $SECRET_NAME \
                    --docker-server=$ECR_REGISTRY \
                    --docker-username=AWS \
                    --docker-password=$TOKEN \
                    --namespace=$NAMESPACE \
                    --dry-run=client -o yaml | kubectl apply -f -
```

## Step 4: Use IRSA (IAM Roles for Service Accounts) on EKS

The best approach for EKS clusters is using IAM Roles for Service Accounts:

```bash
# Create IAM policy for ECR access
cat > ecr-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ECRReadPolicy \
  --policy-document file://ecr-policy.json

# Create IAM role with IRSA trust policy
eksctl create iamserviceaccount \
  --name ecr-puller \
  --namespace production \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::123456789012:policy/ECRReadPolicy \
  --approve
```

## Step 5: Configure ECR in Rancher for EKS Clusters

When Rancher manages EKS clusters, you can configure ECR at the cluster level:

```yaml
# eks-cluster-config.yaml - EKS cluster with ECR access
apiVersion: eks.cattle.io/v1
kind: EKSClusterConfig
metadata:
  name: my-eks-cluster
spec:
  region: us-east-1
  nodeGroups:
    - nodegroupName: default
      instanceType: m5.large
      desiredSize: 3
      # Attach the ECR policy to node IAM role
      nodeRole: arn:aws:iam::123456789012:role/eks-node-role
```

## Step 6: Deploy a Workload Using ECR Image

```yaml
# app-deployment.yaml - Application using ECR image
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      imagePullSecrets:
        - name: ecr-credentials
      containers:
        - name: my-app
          # Full ECR image URI
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

## Step 7: ECR Lifecycle Policies

Manage ECR image lifecycle to control storage costs:

```bash
# Create lifecycle policy to keep only the last 10 images
aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 10 images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["v"],
          "countType": "imageCountMoreThan",
          "countNumber": 10
        },
        "action": {"type": "expire"}
      }
    ]
  }'
```

## Troubleshooting

```bash
# Check if node has ECR permissions
aws sts get-caller-identity

# Verify ECR authentication works
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Debug pod image pull failures
kubectl describe pod <pod-name> -n production | grep -A 10 Events
```

## Conclusion

Integrating AWS ECR with Rancher requires handling the short-lived authentication tokens that ECR uses. For production environments on EKS, use IRSA for seamless authentication. For other cluster types, implement the CronJob-based token refresh pattern to ensure uninterrupted image pulls. This approach provides secure, scalable container image management within your AWS infrastructure.
