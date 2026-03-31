# How to Use ClickHouse with Preset

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Preset, Business Intelligence, Analytics, Visualization

Description: Learn how to connect Preset (managed Apache Superset) to ClickHouse, create certified datasets, build charts, and collaborate on dashboards in the cloud.

---

> Preset is a managed Apache Superset platform that connects to ClickHouse via the clickhouse-connect driver and provides a hosted environment for collaborative analytics.

Preset is the fully managed version of Apache Superset, eliminating infrastructure overhead while providing the full Superset feature set. Since Preset runs on managed infrastructure, connecting it to ClickHouse requires either a public ClickHouse endpoint or configuring network access. This guide covers workspace setup, ClickHouse connection configuration, and building production-grade dashboards.

---

## Setting Up a Preset Workspace

Create a Preset account and workspace.

```text
1. Sign up at https://preset.io
2. Create a new Team workspace
3. Select your workspace region (choose closest to your ClickHouse cluster)
4. Navigate to: Settings -> Database Connections
```

## Preparing ClickHouse for Preset

Configure ClickHouse to accept connections from Preset's IP ranges.

```sql
-- Create a dedicated Preset user
CREATE USER preset_user
IDENTIFIED WITH plaintext_password BY 'preset_strong_password'
HOST ANY;  -- Or restrict to Preset's IP ranges

-- Grant SELECT on your analytics database
GRANT SELECT ON analytics.* TO preset_user;

-- Apply resource limits
CREATE SETTINGS PROFILE preset_profile
    SETTINGS
        max_execution_time  = 90,
        max_rows_to_read    = 500000000,
        use_query_cache     = 1,
        query_cache_ttl     = 600,
        max_memory_usage    = 8000000000;

ALTER USER preset_user SETTINGS PROFILE preset_profile;
```

```bash
# Allow Preset's IP ranges (add to ClickHouse user config if needed)
# Preset publishes their IP ranges at https://docs.preset.io/docs/ip-allowlist

# Example ClickHouse users.xml configuration for IP restriction
cat >> /etc/clickhouse-server/users.d/preset.xml << 'EOF'
<clickhouse>
    <users>
        <preset_user>
            <networks>
                <ip>34.72.0.0/13</ip>
                <ip>34.80.0.0/12</ip>
            </networks>
        </preset_user>
    </users>
</clickhouse>
EOF
```

## Connecting Preset to ClickHouse

Add the ClickHouse database connection in Preset.

```text
Preset Workspace -> Settings -> Database Connections -> + Database

Database Engine: ClickHouse Connect (clickhouse-connect)

Display Name:  ClickHouse Production

SQLAlchemy URI:
  clickhousedb://preset_user:preset_strong_password@clickhouse.example.com:8443/analytics?secure=true

Advanced:
  Engine Parameters (JSON):
  {
    "connect_args": {
      "secure": true,
      "verify": true,
      "compress": true
    }
  }

SQL Lab settings:
  Expose in SQL Lab: Yes
  Allow DML: No
  Allow this database to be explored: Yes
```

## Creating Tables for Preset Dashboards

Build well-structured ClickHouse tables for Preset to explore.

```sql
CREATE TABLE analytics.ecommerce_orders
(
    order_date    Date,
    order_id      UUID,
    customer_id   UInt64,
    country       LowCardinality(String),
    city          String,
    category      LowCardinality(String),
    product_name  String,
    quantity      UInt32,
    unit_price    Float64,
    total_amount  Float64,
    discount_pct  Float32,
    status        LowCardinality(String),
    channel       LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(order_date)
ORDER BY (order_date, country, category, customer_id);

-- Insert sample data
INSERT INTO analytics.ecommerce_orders
SELECT
    today() - (number % 365)                                           AS order_date,
    generateUUIDv4()                                                   AS order_id,
    number % 50000                                                     AS customer_id,
    ['US','UK','DE','FR','JP','CA','AU'][1 + number % 7]               AS country,
    concat('City-', toString(number % 300))                            AS city,
    ['Electronics','Clothing','Home','Sports','Books'][1 + number % 5] AS category,
    concat('Product-', toString(number % 1000))                        AS product_name,
    (rand() % 5) + 1                                                   AS quantity,
    round(randCanonical() * 500 + 10, 2)                               AS unit_price,
    round(randCanonical() * 2500 + 10, 2)                              AS total_amount,
    round(randCanonical() * 30, 1)                                     AS discount_pct,
    ['delivered','pending','shipped','cancelled'][1 + number % 4]      AS status,
    ['web','mobile','email','affiliate'][1 + number % 4]               AS channel
FROM numbers(1000000);
```

## Creating Certified Datasets in Preset

Register and certify datasets for your team to discover.

```text
Preset -> Data -> Datasets -> + Dataset

Database:  ClickHouse Production
Schema:    analytics
Table:     ecommerce_orders

After creating the dataset:
  Edit Dataset -> Settings

  Description: E-commerce orders with customer, product, and channel dimensions.

  Certification:
    Certified: Yes
    Certified by: Data Engineering Team
    Details: Refreshed daily at 2am UTC. Source: Production transactional database.

  Main datetime column: order_date
  Default query limit: 10,000
```

## Building Charts in Preset

Create charts using Preset's drag-and-drop chart builder.

```text
Preset -> Charts -> + Chart

Dataset: ecommerce_orders

Chart 1: Revenue Trend
  Chart type: Line Chart
  X-axis: order_date (by Week)
  Metrics: SUM(total_amount) - label: "Revenue"
  Breakdown: channel
  Filters: status != 'cancelled'
  Rolling window: 4-week moving average

Chart 2: Sales by Country
  Chart type: World Map
  Entity: country
  Metric: SUM(total_amount)
  Color scheme: Blue

Chart 3: Category Performance
  Chart type: Bar Chart (Horizontal)
  Y-axis: category
  Metrics:
    - SUM(total_amount) as Revenue
    - COUNT(*) as Orders
  Sort by: Revenue DESC
```

## Writing SQL in Preset SQL Lab

Use Preset's SQL Lab for ad-hoc ClickHouse queries.

```sql
-- Cohort retention analysis
WITH first_order AS (
    SELECT
        customer_id,
        min(order_date) AS cohort_date
    FROM analytics.ecommerce_orders
    WHERE status != 'cancelled'
    GROUP BY customer_id
),
monthly_activity AS (
    SELECT
        o.customer_id,
        toStartOfMonth(fo.cohort_date)  AS cohort_month,
        toStartOfMonth(o.order_date)    AS activity_month,
        dateDiff('month',
            toStartOfMonth(fo.cohort_date),
            toStartOfMonth(o.order_date)
        )                               AS months_since_first
    FROM analytics.ecommerce_orders o
    JOIN first_order fo ON o.customer_id = fo.customer_id
    WHERE o.status != 'cancelled'
)
SELECT
    toString(cohort_month)                      AS cohort,
    months_since_first,
    count(DISTINCT customer_id)                 AS retained_customers
FROM monthly_activity
GROUP BY cohort_month, months_since_first
ORDER BY cohort_month, months_since_first;
```

## Building a Preset Dashboard

Assemble charts into a collaborative dashboard.

```text
Preset -> Dashboards -> + Dashboard

Name: E-commerce Executive Dashboard

Layout:
  Row 1: KPI scorecards (3 columns)
    - Total Revenue (last 30 days)
    - Total Orders
    - Average Order Value

  Row 2: Revenue Trend (full width)

  Row 3 (2 columns):
    - Sales by Country (Map)
    - Category Performance (Bar)

  Row 4: Recent Orders table

Dashboard filters:
  - Date Range -> connected to all charts
  - Country    -> connected to country-aware charts
  - Channel    -> connected to channel-aware charts

Access control:
  Role-based: Analysts (view only), Data Team (edit)
  Embed: Generate embed link for internal portal
```

## Using Preset's API

Automate dataset and chart management via the Preset API.

```python
import requests

PRESET_URL    = "https://manage.app.preset.io"
PRESET_TOKEN  = "your_api_token"  # from Preset Settings -> API Keys

headers = {
    "Authorization": f"Bearer {PRESET_TOKEN}",
    "Content-Type":  "application/json"
}

# List databases
resp = requests.get(
    f"{PRESET_URL}/api/v1/teams/your-team/workspaces/your-workspace/databases/",
    headers=headers
)
databases = resp.json()
print("Databases:", [d["database_name"] for d in databases["result"]])

# Refresh a dataset
requests.put(
    f"{PRESET_URL}/api/v1/teams/your-team/workspaces/your-workspace/datasets/42/refresh",
    headers=headers
)
```

## Summary

Preset provides a managed Superset environment that connects to ClickHouse via the `clickhouse-connect` SQLAlchemy driver. Create a dedicated ClickHouse user with resource limits, configure the SQLAlchemy URI with SSL enabled, and certify your ClickHouse datasets so team members can discover and trust them. Use the no-code chart builder for standard visualizations and SQL Lab for ad-hoc ClickHouse queries with full access to ClickHouse-specific functions. Role-based access control and embeddable dashboards make Preset suitable for sharing analytics across the organization.
