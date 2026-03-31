# How to Deploy ClickHouse on Google Cloud Compute Engine

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Google Cloud, Compute Engine, Deployment, Linux

Description: Deploy ClickHouse on Google Cloud Compute Engine with optimal machine type selection, persistent disk configuration, firewall rules, and OS tuning.

---

Google Cloud Compute Engine gives you full control over ClickHouse deployments with flexible machine types and high-performance storage options. This guide covers the complete setup from VM creation to production configuration.

## Selecting a Machine Type

For ClickHouse, prefer high-memory or balanced machine types:

- **n2-highmem-16** (16 vCPU, 128 GB) - Good general-purpose choice
- **n2-standard-32** (32 vCPU, 128 GB) - CPU-heavy query workloads
- **m3-ultramem-32** (32 vCPU, 976 GB) - Very large in-memory datasets
- **c3-standard-8-lssd** (8 vCPU, Local SSD) - Lowest latency storage

## Creating the VM

```bash
gcloud compute instances create clickhouse-prod \
  --machine-type=n2-highmem-16 \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=100GB \
  --boot-disk-type=pd-ssd \
  --create-disk=name=clickhouse-data,size=1000GB,type=pd-ssd \
  --zone=us-central1-a \
  --tags=clickhouse-server \
  --scopes=storage-rw
```

The `--scopes=storage-rw` flag allows the VM to write backups to GCS without service account keys.

## Installing ClickHouse

```bash
# SSH into the instance
gcloud compute ssh clickhouse-prod --zone=us-central1-a

# Install ClickHouse on Debian
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/deb/archive/apt/stable.sources' | \
  sudo tee /etc/apt/sources.list.d/clickhouse.sources
sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client
```

## Mounting the Data Disk

```bash
sudo mkfs.ext4 -L clickhouse-data /dev/sdb
sudo mkdir -p /var/lib/clickhouse
sudo mount /dev/sdb /var/lib/clickhouse
sudo chown -R clickhouse:clickhouse /var/lib/clickhouse

# Persist the mount
echo 'LABEL=clickhouse-data /var/lib/clickhouse ext4 defaults,noatime 0 2' | \
  sudo tee -a /etc/fstab
```

## Firewall Rules

```bash
gcloud compute firewall-rules create allow-clickhouse \
  --allow=tcp:8123,tcp:9000 \
  --source-ranges=10.0.0.0/8 \
  --target-tags=clickhouse-server \
  --description="Allow ClickHouse HTTP and native TCP from internal networks"
```

Restrict `source-ranges` to your application subnet only.

## OS Performance Tuning

```bash
# Disable transparent huge pages (persistent via rc.local or systemd)
echo 'never' | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo 'never' | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

# Increase file descriptor limits
echo '* soft nofile 262144' | sudo tee -a /etc/security/limits.conf
echo '* hard nofile 262144' | sudo tee -a /etc/security/limits.conf
```

## Backup to Google Cloud Storage

Use ClickHouse's built-in GCS backup support:

```sql
BACKUP DATABASE mydb
TO GCS('https://storage.googleapis.com/my-backups/clickhouse/', '')
SETTINGS gcs_truncate_on_insert = 1;
```

The VM's service account handles authentication automatically when you attach the right IAM role (`roles/storage.objectAdmin`).

## Summary

Deploying ClickHouse on Google Cloud Compute Engine requires choosing a high-memory machine type, attaching a dedicated persistent SSD for data, configuring XFS or ext4 with noatime, restricting firewall rules to internal networks, and leveraging the VM service account for passwordless GCS backup integration.
