# How to Use ClickHouse with Fivetran

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Fivetran, ELT, Data Integration, Connector, Replication

Description: Configure Fivetran to replicate data from SaaS sources and databases directly into ClickHouse for fast analytical queries with automated sync.

---

Fivetran automates data pipeline setup by providing pre-built connectors that replicate data from hundreds of sources into your data warehouse. ClickHouse is a supported Fivetran destination, letting you continuously sync data from Salesforce, Postgres, Google Analytics, and more.

## Enabling the ClickHouse Destination in Fivetran

1. Log into Fivetran and go to Destinations.
2. Click "Add destination" and search for ClickHouse.
3. Enter your connection details:
   - Host: your ClickHouse hostname or IP
   - Port: 8443 (HTTPS) or 8123 (HTTP)
   - Database: the target database name
   - User: your ClickHouse username
   - Password: your ClickHouse password
4. Click "Save and Test" to verify connectivity.

## Configuring ClickHouse for Fivetran

Fivetran requires the ClickHouse user to have DDL permissions to create and modify tables.

```sql
CREATE USER fivetran IDENTIFIED BY 'strong_password';
GRANT CREATE TABLE, ALTER TABLE, INSERT, SELECT, DROP TABLE ON fivetran_db.* TO fivetran;
```

Enable the HTTP interface in ClickHouse config if it is not already active:

```xml
<http_port>8123</http_port>
<https_port>8443</https_port>
```

## Setting Up a Source Connector

1. In Fivetran, go to Connectors and click "Add connector."
2. Select your source (for example, PostgreSQL, Stripe, or HubSpot).
3. Configure the source connection credentials.
4. Choose your ClickHouse destination.
5. Set the target schema name in ClickHouse.

## Understanding How Fivetran Writes to ClickHouse

Fivetran creates tables automatically in your destination database and handles:

- Initial historical sync (full table load)
- Incremental updates based on change data capture or timestamp columns
- Schema migrations when the source schema changes

Example table created by Fivetran in ClickHouse:

```sql
DESCRIBE TABLE fivetran_db.stripe_charges;
```

```text
id          String
amount      Int64
currency    String
status      String
created     DateTime
_fivetran_synced  DateTime
```

## Querying Replicated Data

After sync completes, query replicated data directly in ClickHouse:

```sql
SELECT
    toStartOfMonth(created) AS month,
    currency,
    sum(amount) / 100 AS total_revenue
FROM fivetran_db.stripe_charges
WHERE status = 'succeeded'
GROUP BY month, currency
ORDER BY month DESC;
```

## Handling ClickHouse-Specific Considerations

ClickHouse uses eventual consistency with the MergeTree engine. Fivetran typically uses ReplacingMergeTree to handle upserts.

```sql
-- Fivetran-managed table structure
CREATE TABLE fivetran_db.users (
    id String,
    email String,
    updated_at DateTime,
    _fivetran_deleted Bool
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY id;
```

## Summary

Fivetran simplifies data replication into ClickHouse by handling connector setup, schema management, and incremental syncs automatically. Once configured, your ClickHouse instance receives a continuous stream of replicated data from all connected sources, ready for fast analytical queries.
