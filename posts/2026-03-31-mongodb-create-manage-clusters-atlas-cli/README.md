# How to Create and Manage Clusters with the Atlas CLI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, CLI, Cluster, DevOps

Description: Learn how to create, configure, scale, pause, and delete MongoDB Atlas clusters entirely from the command line using the Atlas CLI.

---

## Why Manage Clusters from the CLI?

Managing Atlas clusters via the CLI enables repeatable infrastructure operations. You can script cluster creation, embed it in Terraform modules or CI pipelines, and avoid clicking through the portal for routine tasks like scaling or pausing development environments.

## Listing Existing Clusters

Before creating a cluster, review what already exists in your project:

```bash
atlas clusters list
atlas clusters list --output json
```

## Creating a Free Tier Cluster

The quickest way to spin up a development cluster:

```bash
atlas clusters create myFreeCluster \
  --tier M0 \
  --provider AWS \
  --region US_EAST_1
```

M0 is the free shared tier. It does not support all configuration options, but is sufficient for development and testing.

## Creating a Dedicated Cluster

For production workloads, specify a paid tier with backup and version options:

```bash
atlas clusters create myProdCluster \
  --tier M30 \
  --provider AWS \
  --region US_EAST_1 \
  --members 3 \
  --mdbVersion 7.0 \
  --backup \
  --diskSizeGB 100
```

## Using a Cluster Configuration File

For complex cluster configurations, use a JSON spec file:

```json
{
  "name": "myCluster",
  "clusterType": "REPLICASET",
  "providerSettings": {
    "providerName": "AWS",
    "regionName": "US_EAST_1",
    "instanceSizeName": "M30"
  },
  "mongoDBMajorVersion": "7.0",
  "numShards": 1,
  "replicationFactor": 3,
  "backupEnabled": true
}
```

```bash
atlas clusters create --file cluster-config.json
```

## Watching Cluster Creation Progress

Cluster provisioning takes a few minutes. Watch the status in real time:

```bash
atlas clusters watch myProdCluster
```

The command exits once the cluster reaches the IDLE state.

## Scaling a Cluster

Upgrade the instance size of an existing cluster:

```bash
atlas clusters update myProdCluster --tier M40
```

Increase disk size without changing tier:

```bash
atlas clusters update myProdCluster --diskSizeGB 200
```

## Pausing and Resuming Clusters

Pause a development cluster to stop billing compute charges:

```bash
atlas clusters pause myFreeCluster
atlas clusters resume myFreeCluster
```

Note that Atlas pauses clusters automatically after 60 days of inactivity on free tiers.

## Getting the Connection String

Retrieve the SRV connection string to use in your application:

```bash
atlas clusters connectionStrings describe myProdCluster
```

## Deleting a Cluster

Remove a cluster permanently (this is irreversible):

```bash
atlas clusters delete myFreeCluster --force
```

Omit `--force` to get an interactive confirmation prompt.

## Automating Cluster Lifecycle in Scripts

Combine creation, watching, and connection string retrieval in a single script:

```bash
#!/bin/bash
CLUSTER_NAME="ci-test-cluster"

atlas clusters create "$CLUSTER_NAME" \
  --tier M10 \
  --provider AWS \
  --region US_EAST_1

atlas clusters watch "$CLUSTER_NAME"

atlas clusters connectionStrings describe "$CLUSTER_NAME" \
  --output json | jq -r '.standardSrv'
```

## Summary

The Atlas CLI provides full cluster lifecycle management from the command line. Use `atlas clusters create` with a JSON spec file for reproducible cluster definitions, `atlas clusters watch` to block until ready, and `atlas clusters pause` to save costs on idle development clusters.
