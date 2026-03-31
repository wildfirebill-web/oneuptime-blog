# How to Deploy MongoDB on IBM Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, IBM Cloud, Virtual Server, Deployment, Infrastructure

Description: Learn how to deploy MongoDB on IBM Cloud Virtual Server Instances with block storage, security groups, and authentication configuration.

---

## Introduction

IBM Cloud offers Virtual Server Instances (VSI) for classic infrastructure and VPC that can host self-managed MongoDB. This is useful when your workload benefits from IBM Cloud's global data centers, compliance certifications, or proximity to other IBM services like Db2 or Watson.

## Create a VPC Virtual Server Instance

Use the IBM Cloud CLI to create a VSI:

```bash
ibmcloud is instance-create mongodb-primary \
  $VPC_ID \
  us-south-1 \
  bx2-4x16 \
  $SUBNET_ID \
  --image-id $UBUNTU_2204_IMAGE_ID \
  --key-ids $SSH_KEY_ID \
  --primary-network-interface '{"name":"eth0","subnet":{"id":"'$SUBNET_ID'"}}'
```

The instance uses a private subnet. Access it via a VPN gateway or bastion host.

## Create and Attach Block Storage

```bash
ibmcloud is volume-create mongodb-data general-purpose us-south-1 \
  --capacity 200

ibmcloud is instance-volume-attachment-add \
  mongodb-primary-attachment \
  $INSTANCE_ID \
  $VOLUME_ID \
  --auto-delete false
```

## Prepare Storage and Install MongoDB

SSH into the instance and configure the volume:

```bash
sudo mkfs.xfs /dev/vdb
sudo mkdir -p /data/mongodb
sudo mount /dev/vdb /data/mongodb
echo '/dev/vdb /data/mongodb xfs defaults,noatime 0 2' | sudo tee -a /etc/fstab

sudo groupadd mongodb
sudo useradd -g mongodb -s /bin/false mongodb
sudo chown -R mongodb:mongodb /data/mongodb
```

Install MongoDB 7.0:

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

sudo apt-get update && sudo apt-get install -y mongodb-org
```

## Configure MongoDB

Edit `/etc/mongod.conf`:

```yaml
storage:
  dbPath: /data/mongodb
  engine: wiredTiger

net:
  port: 27017
  bindIp: 127.0.0.1,10.240.0.5

security:
  authorization: enabled
```

## Configure Security Group

Create an IBM Cloud VPC security group rule to allow MongoDB traffic:

```bash
ibmcloud is security-group-rule-add $SECURITY_GROUP_ID \
  inbound \
  tcp \
  --port-min 27017 \
  --port-max 27017 \
  --remote $APP_SECURITY_GROUP_ID
```

Using a security group reference as the `--remote` value means only instances in that security group can reach MongoDB.

## Start MongoDB and Create Admin User

```bash
sudo systemctl start mongod && sudo systemctl enable mongod

mongosh admin --eval "
db.createUser({
  user: 'admin',
  pwd: 'IBMCloudMongoPassword!',
  roles: [{ role: 'root', db: 'admin' }]
});
"
```

## Using IBM Cloud Monitoring

Enable IBM Cloud Monitoring to track MongoDB metrics. Install the monitoring agent:

```bash
curl -sL https://ibm.biz/install-sysdig-agent | sudo bash -s -- \
  --access_key $MONITORING_ACCESS_KEY \
  --collector ingest.us-south.monitoring.cloud.ibm.com
```

Configure the MongoDB integration in the Sysdig agent config to collect connection counts, operation rates, and replication lag.

## Summary

Deploying MongoDB on IBM Cloud uses VPC Virtual Server Instances with attached block storage volumes formatted as XFS. Security group rules restrict port 27017 access to specific application server security groups. Enable IBM Cloud Monitoring for operational visibility. For production, deploy a three-node replica set and use IBM Cloud File Storage or Block Storage snapshots for regular backups.
