# How to Create DigitalOcean Managed Databases with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, DigitalOcean, Managed Databases, PostgreSQL, Infrastructure as Code

Description: Learn how to create DigitalOcean managed database clusters with OpenTofu, including PostgreSQL, MySQL, and Redis configurations.

DigitalOcean managed databases handle backups, failover, and patching automatically. OpenTofu lets you define database clusters, users, databases, and connection pools as code.

## Creating a PostgreSQL Cluster

```hcl
resource "digitalocean_database_cluster" "postgres" {
  name       = "production-postgres"
  engine     = "pg"
  version    = "16"
  size       = "db-s-1vcpu-1gb"
  region     = "nyc3"
  node_count = 1  # Use 3 for high availability

  tags = ["production", "database"]
}

output "db_host" {
  value     = digitalocean_database_cluster.postgres.host
  sensitive = true
}

output "db_port" {
  value = digitalocean_database_cluster.postgres.port
}

output "db_uri" {
  value     = digitalocean_database_cluster.postgres.uri
  sensitive = true
}
```

## Creating a Database and User

```hcl
# Create a database within the cluster
resource "digitalocean_database_db" "app" {
  cluster_id = digitalocean_database_cluster.postgres.id
  name       = "appdb"
}

# Create an application user
resource "digitalocean_database_user" "app" {
  cluster_id = digitalocean_database_cluster.postgres.id
  name       = "appuser"
}

output "db_password" {
  value     = digitalocean_database_user.app.password
  sensitive = true
}
```

## Restricting Access with Firewall Rules

```hcl
# Allow only Droplets with the "backend" tag to connect
resource "digitalocean_database_firewall" "postgres" {
  cluster_id = digitalocean_database_cluster.postgres.id

  rule {
    type  = "tag"
    value = "backend"
  }

  # Also allow a specific IP (e.g., bastion host)
  rule {
    type  = "ip_addr"
    value = "203.0.113.10"
  }
}
```

## Creating a Connection Pool

```hcl
resource "digitalocean_database_connection_pool" "app" {
  cluster_id = digitalocean_database_cluster.postgres.id
  name       = "app-pool"
  mode       = "transaction"  # transaction or session
  size       = 10
  db_name    = digitalocean_database_db.app.name
  user       = digitalocean_database_user.app.name
}
```

## Creating a Redis Cluster

```hcl
resource "digitalocean_database_cluster" "redis" {
  name       = "production-redis"
  engine     = "redis"
  version    = "7"
  size       = "db-s-1vcpu-1gb"
  region     = "nyc3"
  node_count = 1

  tags = ["production", "cache"]
}
```

## Creating a MySQL Cluster

```hcl
resource "digitalocean_database_cluster" "mysql" {
  name       = "production-mysql"
  engine     = "mysql"
  version    = "8"
  size       = "db-s-2vcpu-4gb"
  region     = "nyc3"
  node_count = 1
}
```

## Conclusion

DigitalOcean managed databases are straightforward to provision with OpenTofu. Define the cluster, create databases and users within it, add firewall rules to restrict access, and use connection pools for efficient connection management. Store all sensitive outputs like URIs and passwords as sensitive and retrieve them from your application configuration at runtime.
