# How to Configure Sync Modules for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Sync, Multisite, Object Storage, Replication

Description: Learn how to configure and use Ceph RGW sync modules to customize data replication behavior in multisite deployments.

---

Ceph RGW sync modules extend the multisite replication framework to allow custom handling of data as it is replicated between zones. Instead of simply copying objects, sync modules can forward data to external systems, archive it, or apply transformations.

## Built-in Sync Module Types

Ceph RGW ships with several sync module types:

- `default`: Standard object replication between zones
- `archive`: Retains object history indefinitely (even after deletion)
- `cloud-s3`: Replicates data to an external S3-compatible target
- `pubsub`: Publishes bucket notifications to messaging systems
- `elasticsearch`: Indexes object metadata into Elasticsearch

## Setting Up a Zone with an Archive Sync Module

Create a zone group and zone that uses the archive module to preserve all object versions:

```bash
# Create zone group
radosgw-admin zonegroup create \
  --rgw-zonegroup us \
  --endpoints http://primary-rgw:7480 \
  --master --default

# Create primary zone
radosgw-admin zone create \
  --rgw-zonegroup us \
  --rgw-zone us-east-1 \
  --master --default \
  --endpoints http://primary-rgw:7480

# Create archive zone
radosgw-admin zone create \
  --rgw-zonegroup us \
  --rgw-zone us-archive \
  --sync-from us-east-1 \
  --tier-type archive \
  --endpoints http://archive-rgw:7480
```

## Configuring a Cloud-S3 Sync Module

To replicate objects to an external S3 bucket (e.g., AWS):

```bash
radosgw-admin zone modify \
  --rgw-zone cloud-backup \
  --tier-type cloud-s3

radosgw-admin zone placement modify \
  --rgw-zone cloud-backup \
  --placement-id default-placement \
  --tier-type cloud-s3 \
  --tier-config=endpoint=https://s3.amazonaws.com,\
access_key=AKIAIOSFODNN7EXAMPLE,\
secret=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY,\
target_path=my-aws-backup-bucket
```

## Configuring a PubSub Sync Module

The pubsub module publishes object events to an AMQP or Kafka broker:

```bash
radosgw-admin zone modify \
  --rgw-zone events-zone \
  --tier-type pubsub

radosgw-admin zone placement modify \
  --rgw-zone events-zone \
  --placement-id default-placement \
  --tier-type pubsub \
  --tier-config=endpoint=amqp://rabbitmq.example.com,\
exchange=rgw-events
```

## Verifying Sync Status

Check the sync status between zones:

```bash
radosgw-admin sync status
```

For a specific zone:

```bash
radosgw-admin sync status --rgw-zone us-archive
```

## Updating Period After Changes

After any zone or zonegroup changes, commit the period:

```bash
radosgw-admin period update --commit
```

## Summary

Ceph RGW sync modules allow you to customize what happens when objects are replicated across zones. Use the archive module for immutable history, the cloud-s3 module to offload to public cloud, or pubsub to stream events to message brokers. Always commit the period after configuration changes to propagate them across the cluster.
