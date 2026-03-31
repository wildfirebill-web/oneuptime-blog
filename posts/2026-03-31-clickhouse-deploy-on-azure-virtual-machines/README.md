# How to Deploy ClickHouse on Azure Virtual Machines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Azure, Virtual Machine, Deployment, Cloud, Infrastructure, Linux

Description: Deploy a production-ready ClickHouse instance on Azure Virtual Machines with recommended VM sizes, disk configuration, and network security rules.

---

Azure Virtual Machines give you full control over ClickHouse configuration - ideal when managed services don't fit your requirements for version pinning, custom storage layouts, or compliance. This guide walks through VM selection, disk setup, ClickHouse installation, and basic hardening.

## Recommended VM SKUs

| Workload | VM SKU | vCPU | RAM | Notes |
|---|---|---|---|---|
| Development | Standard_D4s_v5 | 4 | 16 GB | Cost-efficient for testing |
| Production (medium) | Standard_E16s_v5 | 16 | 128 GB | Memory-optimized for large queries |
| Production (large) | Standard_E32s_v5 | 32 | 256 GB | Scale up before sharding |
| Compute-heavy | Standard_F32s_v2 | 32 | 64 GB | CPU-bound aggregation workloads |

Use **Premium SSD v2** or **Ultra Disk** for the data volume to match ClickHouse's I/O requirements.

## Provision the VM with Azure CLI

```bash
# Variables
RESOURCE_GROUP="clickhouse-rg"
LOCATION="eastus"
VM_NAME="clickhouse-prod"
VM_SIZE="Standard_E16s_v5"
ADMIN_USER="azureuser"
IMAGE="Canonical:0001-com-ubuntu-server-jammy:22_04-lts-gen2:latest"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create VM
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --size $VM_SIZE \
  --image $IMAGE \
  --admin-username $ADMIN_USER \
  --generate-ssh-keys \
  --os-disk-size-gb 64 \
  --os-disk-caching ReadWrite

# Attach a Premium SSD v2 data disk (1 TB)
az vm disk attach \
  --resource-group $RESOURCE_GROUP \
  --vm-name $VM_NAME \
  --name clickhouse-data-disk \
  --new \
  --size-gb 1024 \
  --sku PremiumV2_LRS \
  --lun 0
```

## Format and Mount the Data Disk

```bash
# SSH into the VM, then:
DISK_DEVICE="/dev/sdc"

# Partition
sudo parted $DISK_DEVICE --script mklabel gpt mkpart primary 0% 100%
sudo mkfs.ext4 -F ${DISK_DEVICE}1

# Mount
sudo mkdir -p /data/clickhouse
echo "${DISK_DEVICE}1 /data/clickhouse ext4 defaults,noatime,nodiratime 0 2" | sudo tee -a /etc/fstab
sudo mount -a
```

## Install ClickHouse

```bash
# Add ClickHouse repository
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | sudo gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] https://packages.clickhouse.com/deb stable main" | sudo tee /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update

# Install
sudo apt-get install -y clickhouse-server clickhouse-client

# Move data directory to the mounted disk
sudo systemctl stop clickhouse-server
sudo mv /var/lib/clickhouse /data/clickhouse/data
sudo ln -s /data/clickhouse/data /var/lib/clickhouse
sudo chown -R clickhouse:clickhouse /data/clickhouse

sudo systemctl enable clickhouse-server
sudo systemctl start clickhouse-server
```

## Configure Network Security Group Rules

```bash
# Allow ClickHouse HTTP interface from specific CIDR only
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name "${VM_NAME}NSG" \
  --name AllowClickHouseHTTP \
  --priority 1100 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-port-ranges 8123 \
  --protocol Tcp \
  --access Allow

# Allow native TCP protocol
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name "${VM_NAME}NSG" \
  --name AllowClickHouseTCP \
  --priority 1200 \
  --source-address-prefixes "10.0.0.0/16" \
  --destination-port-ranges 9000 \
  --protocol Tcp \
  --access Allow
```

## Tune the OS for ClickHouse

```bash
# Increase open file limits
echo "clickhouse soft nofile 1048576" | sudo tee -a /etc/security/limits.conf
echo "clickhouse hard nofile 1048576" | sudo tee -a /etc/security/limits.conf

# Disable transparent huge pages
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

# Persist across reboots
cat <<'EOF' | sudo tee /etc/rc.local
#!/bin/bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
exit 0
EOF
sudo chmod +x /etc/rc.local
```

## Summary

Deploying ClickHouse on Azure VMs gives you full control over storage layout, OS tuning, and ClickHouse version. Use memory-optimized VMs, attach Premium SSD v2 disks for data, restrict network access via NSG rules, and disable transparent huge pages for best performance. Once running, consider configuring ClickHouse Keeper or ZooKeeper for replication if you need high availability.
