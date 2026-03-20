# How to Create OCI Autonomous Databases with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Oracle Cloud, Autonomous Database, OCI, Infrastructure as Code

Description: Learn how to create Oracle Cloud Infrastructure Autonomous Databases with OpenTofu for self-managing, auto-scaling database workloads.

OCI Autonomous Database is a fully managed, self-driving Oracle Database service that automatically handles patching, tuning, and scaling. OpenTofu lets you provision Autonomous Transaction Processing (ATP), Autonomous Data Warehouse (ADW), and JSON Database instances as code.

## Creating an Autonomous Transaction Processing Database

```hcl
resource "oci_database_autonomous_database" "atp" {
  compartment_id           = var.compartment_id
  display_name             = "production-atp"
  db_name                  = "PRODATP"              # Max 14 alphanumeric chars
  admin_password           = var.db_admin_password  # Must meet complexity requirements
  cpu_core_count           = 2                      # OCPUs
  data_storage_size_in_tbs = 1                      # Storage in TBs

  # Workload type: OLTP (ATP), DW (ADW), AJD (JSON), APEX
  db_workload    = "OLTP"
  license_model  = "LICENSE_INCLUDED"  # or BRING_YOUR_OWN_LICENSE
  is_free_tier   = false

  freeform_tags = {
    Environment = "production"
    ManagedBy   = "opentofu"
  }
}

output "atp_connection_strings" {
  value     = oci_database_autonomous_database.atp.connection_strings
  sensitive = true
}
```

## Creating an Always Free Autonomous Database

```hcl
resource "oci_database_autonomous_database" "free" {
  compartment_id = var.compartment_id
  display_name   = "free-atp"
  db_name        = "FREEATP"
  admin_password = var.db_admin_password

  # Free tier limits: 1 OCPU, 20 GB
  cpu_core_count           = 1
  data_storage_size_in_tbs = null
  data_storage_size_in_gbs = 20

  is_free_tier  = true
  db_workload   = "OLTP"
}
```

## Creating an Autonomous Data Warehouse

```hcl
resource "oci_database_autonomous_database" "adw" {
  compartment_id           = var.compartment_id
  display_name             = "analytics-adw"
  db_name                  = "ANALYSADW"
  admin_password           = var.db_admin_password
  cpu_core_count           = 4
  data_storage_size_in_tbs = 2
  db_workload              = "DW"
  license_model            = "LICENSE_INCLUDED"
}
```

## Auto-Scaling

```hcl
resource "oci_database_autonomous_database" "autoscale" {
  # ...
  is_auto_scaling_enabled                = true  # CPU auto-scaling
  is_auto_scaling_for_storage_enabled    = true  # Storage auto-scaling
  cpu_core_count                         = 2     # Base OCPUs (scales to 3x)
  data_storage_size_in_tbs              = 1
}
```

## Configuring Network Access

```hcl
# Allow access from specific IPs (private access endpoint is more secure)

resource "oci_database_autonomous_database" "secure" {
  compartment_id           = var.compartment_id
  display_name             = "secure-atp"
  db_name                  = "SECUREATP"
  admin_password           = var.db_admin_password
  cpu_core_count           = 2
  data_storage_size_in_tbs = 1
  db_workload              = "OLTP"

  # Place in a private subnet (no public endpoint)
  subnet_id    = oci_core_subnet.private.id
  private_endpoint_label = "secureatp"

  is_mtls_connection_required = true  # Require mTLS connections
}
```

## Getting the Wallet for Connection

Download the connection wallet via the CLI after provisioning:

```bash
oci db autonomous-database generate-wallet \
  --autonomous-database-id $(tofu output -raw atp_id) \
  --password "WalletPass123!" \
  --file wallet.zip
```

## Conclusion

OCI Autonomous Database provides a self-managing Oracle Database with zero operational overhead. Use ATP for transactional workloads, ADW for analytics, and the free tier for development. Enable auto-scaling for production workloads, and use private endpoints with mTLS for secure connectivity from OCI resources.
