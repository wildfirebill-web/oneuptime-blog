# How to Deploy ClickHouse on Azure Virtual Machines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Azure, Virtual Machine, Deployment, Linux

Description: Deploy ClickHouse on Azure Virtual Machines with the right VM size, managed disk configuration, network security groups, and Azure Blob backup integration.

---

Azure Virtual Machines offer a flexible foundation for ClickHouse deployments, with a wide range of VM sizes suited to analytics workloads and native integration with Azure Blob Storage for backups.

## Choosing the Right VM Size

Recommended Azure VM series for ClickHouse:

- **Esv5** (e.g., Standard_E16s_v5) - Memory optimized, 128 GB RAM
- **Msv3** - Very large memory for huge datasets
- **Lsv3** - Storage optimized with local NVMe disks
- **Fsv2** - Compute optimized for CPU-heavy query workloads

## Creating the VM with Azure CLI

```bash
az vm create \
  --resource-group clickhouse-rg \
  --name clickhouse-prod \
  --image Ubuntu2204 \
  --size Standard_E16s_v5 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --vnet-name clickhouse-vnet \
  --subnet clickhouse-subnet \
  --public-ip-address "" \
  --output table
```

Use `--public-ip-address ""` to avoid assigning a public IP - connect through a bastion host or VPN.

## Attaching a Managed Disk

```bash
# Create and attach a 1 TB Premium SSD
az disk create \
  --resource-group clickhouse-rg \
  --name clickhouse-data-disk \
  --size-gb 1024 \
  --sku Premium_LRS \
  --zone 1

az vm disk attach \
  --resource-group clickhouse-rg \
  --vm-name clickhouse-prod \
  --name clickhouse-data-disk
```

## Formatting and Mounting the Disk

```bash
# SSH into VM
sudo fdisk -l  # find disk (likely /dev/sdc)
sudo mkfs.xfs /dev/sdc
sudo mkdir -p /var/lib/clickhouse
sudo mount /dev/sdc /var/lib/clickhouse
sudo chown -R clickhouse:clickhouse /var/lib/clickhouse

# Persist
echo '/dev/sdc /var/lib/clickhouse xfs defaults,noatime 0 2' | sudo tee -a /etc/fstab
```

## Installing ClickHouse on Ubuntu

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL 'https://packages.clickhouse.com/deb/archive/apt/stable.sources' | \
  sudo tee /etc/apt/sources.list.d/clickhouse.sources
sudo apt-get update && sudo apt-get install -y clickhouse-server clickhouse-client
sudo systemctl enable --now clickhouse-server
```

## Network Security Group Rules

```bash
az network nsg rule create \
  --resource-group clickhouse-rg \
  --nsg-name clickhouse-nsg \
  --name allow-clickhouse-internal \
  --priority 100 \
  --protocol Tcp \
  --destination-port-ranges 8123 9000 \
  --source-address-prefixes 10.0.0.0/8 \
  --access Allow
```

## Azure Blob Storage Backup

ClickHouse supports Azure Blob Storage natively for backups:

```sql
BACKUP DATABASE mydb
TO AzureBlobStorage(
  'DefaultEndpointsProtocol=https;AccountName=myaccount;AccountKey=xxx;EndpointSuffix=core.windows.net',
  'clickhouse-backups',
  'backups/mydb'
);
```

Use a Managed Identity on the VM to avoid storing the account key in SQL:

```xml
<clickhouse>
  <azure_storage>
    <use_managed_identity>true</use_managed_identity>
  </azure_storage>
</clickhouse>
```

## Summary

Deploying ClickHouse on Azure VMs requires selecting a memory or storage optimized VM size, attaching a Premium SSD managed disk, configuring XFS with noatime, restricting network access to internal IP ranges via NSG rules, and using Managed Identity for secure Azure Blob Storage backup access.
