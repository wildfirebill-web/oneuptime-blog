# How to Use AWS DMS to Migrate to MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, AWS DMS, Migration, Cloud, Replication

Description: Use AWS Database Migration Service to migrate an existing database to MySQL with minimal downtime using full-load and ongoing change-data-capture replication.

---

## What Is AWS DMS?

AWS Database Migration Service (DMS) migrates databases to AWS with minimal downtime. It supports homogeneous migrations (MySQL to MySQL) and heterogeneous migrations (Oracle, SQL Server, PostgreSQL to MySQL). DMS runs a full initial load, then uses ongoing replication (CDC - Change Data Capture) to keep the target in sync until you cut over.

## Architecture Overview

```text
Source DB --> DMS Replication Instance --> Target MySQL (RDS/Aurora)
             [Full Load + CDC]
```

## Step 1 - Create the Target MySQL Database

```bash
aws rds create-db-instance \
  --db-instance-identifier myapp-mysql \
  --db-instance-class db.t3.medium \
  --engine mysql \
  --engine-version 8.0 \
  --master-username admin \
  --master-user-password 'StrongPass!' \
  --allocated-storage 100 \
  --vpc-security-group-ids sg-0abc12345 \
  --db-subnet-group-name my-db-subnet-group
```

## Step 2 - Create a DMS Replication Instance

```bash
aws dms create-replication-instance \
  --replication-instance-identifier my-dms-instance \
  --replication-instance-class dms.t3.medium \
  --allocated-storage 50 \
  --vpc-security-group-ids sg-0abc12345 \
  --replication-subnet-group-identifier my-dms-subnet-group
```

## Step 3 - Create Source and Target Endpoints

```bash
# Source endpoint (e.g., on-premise MySQL)
aws dms create-endpoint \
  --endpoint-identifier source-mysql \
  --endpoint-type source \
  --engine-name mysql \
  --username dms_user \
  --password 'SourcePass!' \
  --server-name 10.0.1.50 \
  --port 3306 \
  --database-name myapp

# Target endpoint (RDS MySQL)
aws dms create-endpoint \
  --endpoint-identifier target-rds-mysql \
  --endpoint-type target \
  --engine-name mysql \
  --username admin \
  --password 'StrongPass!' \
  --server-name myapp-mysql.abc123.us-east-1.rds.amazonaws.com \
  --port 3306 \
  --database-name myapp
```

## Step 4 - Prepare the Source for CDC

For MySQL as source, enable binary logging:

```ini
[mysqld]
log-bin = mysql-bin
binlog-format = ROW
binlog-row-image = FULL
server-id = 1
expire_logs_days = 3
```

Create a DMS user on the source:

```sql
CREATE USER 'dms_user'@'%' IDENTIFIED BY 'DmsPass!';
GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'dms_user'@'%';
GRANT SELECT ON myapp.* TO 'dms_user'@'%';
FLUSH PRIVILEGES;
```

## Step 5 - Create and Start the Replication Task

```bash
aws dms create-replication-task \
  --replication-task-identifier myapp-migration \
  --source-endpoint-arn arn:aws:dms:us-east-1:123456789:endpoint:source-mysql \
  --target-endpoint-arn arn:aws:dms:us-east-1:123456789:endpoint:target-rds-mysql \
  --replication-instance-arn arn:aws:dms:us-east-1:123456789:rep:my-dms-instance \
  --migration-type full-load-and-cdc \
  --table-mappings '{"rules":[{"rule-type":"selection","rule-id":"1","rule-name":"include-all","object-locator":{"schema-name":"myapp","table-name":"%"},"rule-action":"include"}]}'

aws dms start-replication-task \
  --replication-task-arn arn:aws:dms:us-east-1:123456789:task:myapp-migration \
  --start-replication-task-type start-replication
```

## Step 6 - Monitor Progress

```bash
aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=arn:aws:dms:...:task:myapp-migration \
  --query 'ReplicationTasks[0].{Status:Status,Percent:ReplicationTaskStats.FullLoadProgressPercent}'
```

## Step 7 - Cutover

Once CDC lag drops to near zero:

1. Stop writes to the source database
2. Wait for DMS to catch up (lag = 0)
3. Update your application connection string to the new RDS endpoint
4. Resume writes on the target

## Summary

AWS DMS handles database migration to MySQL through a two-phase process: a full initial load followed by continuous CDC replication. This keeps the target in sync with the source until you are ready to cut over with minimal downtime. Ensure binary logging is enabled on the source and the DMS user has replication privileges for CDC to work correctly.
