# How to Deploy Snowflake Resources with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Snowflake, Data Warehouse, Infrastructure as Code, Data Engineering, Analytics

Description: Learn how to manage Snowflake databases, warehouses, roles, and users using OpenTofu for governed, reproducible Snowflake infrastructure management.

---

Snowflake separates compute (virtual warehouses) from storage, making it uniquely flexible. Managing Snowflake resources manually through the UI leads to configuration drift, undocumented grants, and hard-to-debug permission issues. OpenTofu's Snowflake provider brings these resources under version control.

## Provider Configuration

```hcl
# main.tf

terraform {
  required_providers {
    snowflake = {
      source  = "Snowflake-Labs/snowflake"
      version = "~> 0.87"
    }
  }
}

provider "snowflake" {
  account  = var.snowflake_account
  username = var.snowflake_username
  password = var.snowflake_password
  role     = "SYSADMIN"
}
```

## Creating Databases and Schemas

```hcl
# databases.tf
# Create a production database
resource "snowflake_database" "production" {
  name    = "PRODUCTION"
  comment = "Production data warehouse database"

  # Data retention for Time Travel
  data_retention_time_in_days = 7
}

# Create schemas within the database
resource "snowflake_schema" "raw" {
  database = snowflake_database.production.name
  name     = "RAW"
  comment  = "Raw ingested data - do not query directly"
  data_retention_time_in_days = 1  # Short retention for raw zone
}

resource "snowflake_schema" "analytics" {
  database = snowflake_database.production.name
  name     = "ANALYTICS"
  comment  = "Analytics-ready tables and views"
  data_retention_time_in_days = 7
}
```

## Creating Virtual Warehouses

```hcl
# warehouses.tf
# ETL warehouse - suspend when idle to save credits
resource "snowflake_warehouse" "etl" {
  name           = "ETL_WAREHOUSE"
  warehouse_size = "MEDIUM"
  auto_suspend   = 60   # Suspend after 60 seconds of inactivity
  auto_resume    = true
  comment        = "Warehouse for ETL/ELT transformations"

  # Maximize resource for batch jobs
  max_cluster_count = 3
  min_cluster_count = 1
  scaling_policy    = "ECONOMY"
}

# Reporting warehouse - responsive for end users
resource "snowflake_warehouse" "reporting" {
  name           = "REPORTING_WAREHOUSE"
  warehouse_size = "SMALL"
  auto_suspend   = 120  # Users expect faster resume
  auto_resume    = true
  comment        = "Warehouse for BI tools and ad-hoc queries"

  max_cluster_count = 5
  min_cluster_count = 1
  scaling_policy    = "STANDARD"
}
```

## Creating Roles and Granting Permissions

```hcl
# roles.tf
# Create a role for data analysts
resource "snowflake_role" "data_analyst" {
  name    = "DATA_ANALYST"
  comment = "Read access to analytics schema for data analysts"
}

# Grant usage on the database
resource "snowflake_grant_privileges_to_role" "analyst_db_usage" {
  role_name  = snowflake_role.data_analyst.name
  privileges = ["USAGE"]
  on_schema_object {
    object_type = "DATABASE"
    object_name = snowflake_database.production.name
  }
}

# Grant select on all tables in the analytics schema
resource "snowflake_grant_privileges_to_role" "analyst_select" {
  role_name  = snowflake_role.data_analyst.name
  privileges = ["SELECT"]
  on_schema_object {
    all {
      object_type_plural = "TABLES"
      in_schema          = "${snowflake_database.production.name}.${snowflake_schema.analytics.name}"
    }
  }
}

# Grant warehouse usage
resource "snowflake_grant_privileges_to_role" "analyst_warehouse" {
  role_name  = snowflake_role.data_analyst.name
  privileges = ["USAGE"]
  on_schema_object {
    object_type = "WAREHOUSE"
    object_name = snowflake_warehouse.reporting.name
  }
}
```

## Creating Users

```hcl
# users.tf
# Create a service account user for the ETL pipeline
resource "snowflake_user" "etl_service" {
  name         = "ETL_SERVICE_ACCOUNT"
  login_name   = "etl_service"
  password     = var.etl_service_password
  comment      = "Service account for ETL pipeline"
  default_role = "ETL_ROLE"
  default_warehouse = snowflake_warehouse.etl.name

  must_change_password = false
}

resource "snowflake_role_grants" "etl_role_grant" {
  role_name = "ETL_ROLE"
  users     = [snowflake_user.etl_service.name]
}
```

## Best Practices

- Always set `auto_suspend` on warehouses - even 60 seconds of auto-suspend eliminates most idle costs.
- Use Role-Based Access Control (RBAC) with functional roles rather than granting permissions directly to users.
- Set `data_retention_time_in_days = 7` on production databases to enable time travel for data recovery.
- Use Snowflake's `FUTURE GRANTS` where possible to automatically apply grants to new tables - OpenTofu models this with `on_future_schemas_in_database`.
- Store Snowflake credentials in a secrets manager, not in OpenTofu variable files.
