# How to Use Crossplane with Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Crossplane, Infrastructure as Code, Cloud Native

Description: Use Crossplane with Rancher to manage cloud infrastructure resources directly from Kubernetes using CRDs for a fully cloud-native infrastructure management approach.

## Introduction

Crossplane extends Kubernetes to manage cloud infrastructure using the same Kubernetes API and tooling. Instead of using separate tools like Terraform or Pulumi, Crossplane allows you to provision AWS, Azure, GCP, and other cloud resources through Kubernetes manifests. This guide covers installing Crossplane on Rancher and using it to provision cloud resources alongside your Kubernetes workloads.

## Prerequisites

- Rancher-managed Kubernetes cluster
- Helm 3.x installed
- Cloud provider credentials (AWS, Azure, or GCP)
- kubectl access

## Step 1: Install Crossplane

```bash
# Add Crossplane Helm repository

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

# Install Crossplane
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --wait

# Verify installation
kubectl get pods -n crossplane-system
kubectl get crds | grep crossplane.io | head -10
```

## Step 2: Install Cloud Providers

### AWS Provider

```yaml
# aws-provider.yaml - Install AWS Crossplane provider
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.0.0
  controllerConfigRef:
    name: provider-aws-controller-config
---
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: provider-aws-controller-config
spec:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
```

```bash
kubectl apply -f aws-provider.yaml

# Wait for provider to be healthy
kubectl get providers
```

### Configure AWS Credentials

```bash
# Create AWS credentials secret
kubectl create secret generic aws-credentials \
  --from-literal=credentials="[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" \
  -n crossplane-system
```

```yaml
# aws-provider-config.yaml - Provider configuration
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-credentials
      key: credentials
```

## Step 3: Provision AWS Resources

### Create an S3 Bucket

```yaml
# s3-bucket.yaml - Provision S3 bucket via Crossplane
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: my-app-storage
  namespace: production
spec:
  forProvider:
    region: us-east-1
    tags:
      environment: production
      managed-by: crossplane
  providerConfigRef:
    name: default
  writeConnectionSecretToRef:
    namespace: production
    name: s3-connection-details
---
# BucketVersioning
apiVersion: s3.aws.upbound.io/v1beta1
kind: BucketVersioning
metadata:
  name: my-app-storage-versioning
spec:
  forProvider:
    bucketRef:
      name: my-app-storage
    region: us-east-1
    versioningConfiguration:
      - status: Enabled
```

### Create an RDS Database

```yaml
# rds-instance.yaml - Provision RDS PostgreSQL via Crossplane
apiVersion: rds.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: production-db
  namespace: production
spec:
  forProvider:
    region: us-east-1
    dbInstanceClass: db.t3.medium
    engine: postgres
    engineVersion: "16.1"
    allocatedStorage: 20
    maxAllocatedStorage: 100
    storageType: gp3
    multiAz: true
    username: dbadmin
    autoMinorVersionUpgrade: true
    backupRetentionPeriod: 7
    skipFinalSnapshot: false
    finalSnapshotIdentifier: production-db-final
    tags:
      environment: production
  writeConnectionSecretToRef:
    namespace: production
    name: rds-connection
  providerConfigRef:
    name: default
```

## Step 4: Create Composite Resources (XRDs)

Crossplane's power comes from composites that bundle multiple resources:

```yaml
# xrd-database.yaml - Composite Resource Definition
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.database.example.com
spec:
  group: database.example.com
  names:
    kind: XPostgreSQLInstance
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance
    plural: postgresqlinstances
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    storageGB:
                      type: integer
                      description: Size of storage in GB
                    tier:
                      type: string
                      enum: [small, medium, large]
                  required:
                    - storageGB
                    - tier
---
# composition.yaml - Composition (implementation)
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xpostgresqlinstances.aws.database.example.com
  labels:
    provider: aws
spec:
  compositeTypeRef:
    apiVersion: database.example.com/v1alpha1
    kind: XPostgreSQLInstance
  resources:
    - name: rdsInstance
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            region: us-east-1
            engine: postgres
            engineVersion: "16.1"
            multiAz: true
            backupRetentionPeriod: 7
      patches:
        # Map tier to instance class
        - type: CombineFromComposite
          combine:
            variables:
              - fromFieldPath: spec.parameters.tier
          toFieldPath: spec.forProvider.dbInstanceClass
          policy:
            fromFieldPath: Required
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.storageGB
          toFieldPath: spec.forProvider.allocatedStorage
```

## Step 5: Claim Composite Resources

```yaml
# postgres-claim.yaml - Claim a PostgreSQL instance
apiVersion: database.example.com/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: my-app-db
  namespace: production
spec:
  parameters:
    storageGB: 20
    tier: medium
  compositionSelector:
    matchLabels:
      provider: aws
  writeConnectionSecretToRef:
    name: my-app-db-connection
```

## Step 6: Use Crossplane with Rancher Fleet

```yaml
# fleet-crossplane.yaml - Manage Crossplane resources with Fleet
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-claims
  namespace: production
data:
  claim.yaml: |
    apiVersion: database.example.com/v1alpha1
    kind: PostgreSQLInstance
    metadata:
      name: team-db
    spec:
      parameters:
        storageGB: 20
        tier: small
```

## Conclusion

Crossplane brings infrastructure provisioning into the Kubernetes ecosystem, enabling platform teams to provide self-service infrastructure to development teams using familiar Kubernetes tools. Composite Resources abstract cloud-specific details, providing a consistent API regardless of the underlying cloud provider. In Rancher environments, Crossplane pairs well with GitOps workflows through Fleet, enabling teams to request infrastructure resources alongside their application deployments.
