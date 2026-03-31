# How to Use PlanetScale for MySQL in the Cloud

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, PlanetScale, Cloud, Database, Branching

Description: Learn how to use PlanetScale, a MySQL-compatible serverless database platform, to manage schemas with branching and deploy schema changes without downtime.

---

## What Is PlanetScale

PlanetScale is a serverless MySQL-compatible database platform built on Vitess. It adds developer-friendly features on top of MySQL: database branching for schema changes, non-blocking schema migrations using online DDL, and automatic horizontal scaling via Vitess sharding. Applications connect using the standard MySQL protocol.

## Installing the PlanetScale CLI

```bash
# macOS
brew install planetscale/tap/pscale

# Linux
curl -fsSL https://cli.planetscale.com/install.sh | sudo bash
```

Authenticate:

```bash
pscale auth login
```

## Creating a Database

```bash
pscale database create my-app-db --region us-east
```

List your databases:

```bash
pscale database list
```

## Connecting from an Application

PlanetScale uses a proxy for local development. Start a local proxy session:

```bash
pscale connect my-app-db main --port 3309
```

Then connect your application to `127.0.0.1:3309` without TLS. In production, generate a password and connect directly:

```bash
pscale password create my-app-db main prod-password
```

Use the returned credentials in your application's connection string:

```text
mysql://username:password@aws.connect.psdb.cloud/my-app-db?ssl-mode=require
```

## Schema Branching Workflow

PlanetScale's branching model is similar to Git. You create a branch, apply schema changes, then open a deploy request to merge changes into the main branch.

```bash
# Create a development branch
pscale branch create my-app-db add-users-index

# Connect to the branch
pscale connect my-app-db add-users-index --port 3310
```

Apply your DDL on the branch:

```sql
ALTER TABLE users ADD INDEX idx_email (email);
```

Open a deploy request to merge the schema change into `main`:

```bash
pscale deploy-request create my-app-db add-users-index
```

PlanetScale runs the migration using non-blocking online DDL (via `gh-ost` or Vitess), so the `ALTER TABLE` does not lock the table on the production branch.

## Enabling Safe Migrations

Enable safe migrations on the main branch to prevent direct DDL on production:

```bash
pscale branch safe-migrations enable my-app-db main
```

With safe migrations enabled, all schema changes must go through a deploy request.

## Querying with the PlanetScale Shell

```bash
pscale shell my-app-db main
```

This opens an interactive MySQL shell connected to the `main` branch, useful for running queries without setting up a full connection.

## Checking Database Insights

PlanetScale's console at `app.planetscale.com` provides query statistics, slow query logs, and index recommendations. Use the Insights tab to identify queries missing indexes and optimize them before they affect production.

## Summary

PlanetScale brings a Git-like branching model to MySQL schema management, enabling zero-downtime schema changes through deploy requests and online DDL. Use the `pscale` CLI for local development with a proxy connection, generate password credentials for production, and leverage the Insights console to optimize slow queries. PlanetScale is well-suited for teams that need scalable MySQL without managing replication topology themselves.
