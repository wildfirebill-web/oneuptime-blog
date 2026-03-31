# How to Use ClickHouse with Power BI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Power BI, Business Intelligence, Analytics, Visualization

Description: Learn how to connect Power BI Desktop to ClickHouse using the ODBC connector, import data, write DAX measures, and publish reports to Power BI Service.

---

> Power BI connects to ClickHouse via ODBC and enables rich interactive reports on analytical data with scheduled dataset refreshes through the on-premises data gateway.

Power BI is Microsoft's flagship BI platform used by millions of enterprises. Connecting it to ClickHouse enables Power BI reports and dashboards to run on high-performance analytical data. This guide covers ODBC driver setup, DirectQuery vs. Import mode, writing M queries, and publishing to Power BI Service.

---

## Installing the ClickHouse ODBC Driver

Install the ClickHouse ODBC driver on Windows.

```text
1. Download the ClickHouse ODBC driver:
   https://github.com/ClickHouse/clickhouse-odbc/releases

2. Run the installer: clickhouse-odbc-x.x.x-win64.msi

3. Open ODBC Data Sources (64-bit) from Windows administrative tools

4. System DSN -> Add -> ClickHouse ODBC Driver (Unicode)

DSN Name:        ClickHouse_Production
Host:            clickhouse.example.com
Port:            8443
Database:        analytics
UID:             powerbi_user
PWD:             powerbi_password
SSLMode:         require
Timeout:         120
```

## Connecting Power BI to ClickHouse

Open Power BI Desktop and configure the ODBC connection.

```text
Power BI Desktop:
  Home -> Get Data -> ODBC

  Data source name (DSN): ClickHouse_Production

  Advanced Options -> SQL statement (optional):
  SELECT * FROM analytics.sales_summary LIMIT 1000000

  Credentials:
    Windows -> Use my current credentials
    Or: Database -> Enter credentials

  Navigator:
    Select tables to import:
    - analytics.sales_summary
    - analytics.customer_segments
    - analytics.product_catalog
```

## Creating a PowerBI User in ClickHouse

Set up the Power BI user with appropriate permissions.

```sql
-- Create a dedicated Power BI user
CREATE USER powerbi_user
IDENTIFIED WITH plaintext_password BY 'powerbi_password'
HOST ANY;

-- Grant SELECT on analytics tables
GRANT SELECT ON analytics.* TO powerbi_user;

-- Create a resource profile
CREATE SETTINGS PROFILE powerbi_profile
    SETTINGS
        max_execution_time  = 120,
        max_rows_to_read    = 1000000000,
        use_query_cache     = 1,
        query_cache_ttl     = 600;

ALTER USER powerbi_user SETTINGS PROFILE powerbi_profile;
```

## Creating Optimized ClickHouse Tables

Design tables for Power BI Import mode.

```sql
CREATE TABLE analytics.sales_summary
(
    date           Date,
    country        LowCardinality(String),
    region         LowCardinality(String),
    segment        LowCardinality(String),
    category       LowCardinality(String),
    sub_category   LowCardinality(String),
    product_id     UInt32,
    product_name   String,
    total_sales    Float64,
    total_profit   Float64,
    total_units    UInt32,
    order_count    UInt32,
    customer_count UInt32
)
ENGINE = SummingMergeTree(
    (total_sales, total_profit, total_units, order_count, customer_count)
)
PARTITION BY toYYYYMM(date)
ORDER BY (date, country, region, segment, category);

-- Populate with sample data
INSERT INTO analytics.sales_summary
SELECT
    today() - (number % 730)                                      AS date,
    ['US','UK','DE','FR','CA'][1 + number % 5]                    AS country,
    ['North','South','East','West'][1 + number % 4]               AS region,
    ['Consumer','Corporate','Home Office'][1 + number % 3]        AS segment,
    ['Tech','Furniture','Office'][1 + number % 3]                 AS category,
    ['Phones','Desks','Paper','Cables'][1 + number % 4]           AS sub_category,
    (number % 200) + 1                                            AS product_id,
    concat('Product-', toString(number % 200))                    AS product_name,
    round(randCanonical() * 5000, 2)                              AS total_sales,
    round(randCanonical() * 1000 - 100, 2)                        AS total_profit,
    (rand() % 50) + 1                                             AS total_units,
    (rand() % 20) + 1                                             AS order_count,
    (rand() % 15) + 1                                             AS customer_count
FROM numbers(200000);
```

## Writing M Queries (Power Query)

Use the Power Query editor to transform ClickHouse data.

```text
Power Query M formula:
```

```text
let
    Source = Odbc.DataSource(
        "dsn=ClickHouse_Production",
        [
            HierarchicalNavigation = true,
            ConnectionTimeout = #duration(0, 0, 2, 0)
        ]
    ),
    analytics_db = Source{[Name="analytics"]}[Data],
    sales_table  = analytics_db{[Name="sales_summary"]}[Data],

    // Filter to last 2 years
    filtered = Table.SelectRows(
        sales_table,
        each [date] >= Date.AddYears(Date.From(DateTime.LocalNow()), -2)
    ),

    // Add a Year-Month column
    with_ym = Table.AddColumn(
        filtered,
        "YearMonth",
        each Date.ToText([date], "yyyy-MM"),
        type text
    ),

    // Change types
    typed = Table.TransformColumnTypes(
        with_ym,
        {
            {"date",          type date},
            {"total_sales",   type number},
            {"total_profit",  type number},
            {"total_units",   Int64.Type},
            {"order_count",   Int64.Type},
            {"customer_count",Int64.Type}
        }
    )
in
    typed
```

## Writing DAX Measures

Create DAX measures that work with ClickHouse-imported data.

```text
// Total Sales measure
Total Sales =
    SUM(sales_summary[total_sales])

// Profit Margin
Profit Margin % =
    DIVIDE(
        SUM(sales_summary[total_profit]),
        SUM(sales_summary[total_sales]),
        0
    ) * 100

// Year-over-Year growth
YoY Sales Growth % =
    VAR current_year =
        CALCULATE(
            [Total Sales],
            YEAR(sales_summary[date]) = YEAR(TODAY())
        )
    VAR prior_year =
        CALCULATE(
            [Total Sales],
            YEAR(sales_summary[date]) = YEAR(TODAY()) - 1
        )
    RETURN
        DIVIDE(current_year - prior_year, prior_year, 0) * 100

// Running total
Running Total Sales =
    CALCULATE(
        [Total Sales],
        FILTER(
            ALL(sales_summary[date]),
            sales_summary[date] <= MAX(sales_summary[date])
        )
    )
```

## DirectQuery Mode

Use DirectQuery to always query live ClickHouse data.

```text
Power BI Desktop:
  Get Data -> ODBC -> ClickHouse_Production

  At the bottom of the Navigator:
  Select "DirectQuery" instead of "Import"

  Best practices for DirectQuery on ClickHouse:
  - Use pre-aggregated SummingMergeTree tables
  - Limit the number of visuals per page (< 6 recommended)
  - Avoid visuals with row-level detail on large tables
  - Set max rows returned: 1,000,000 (Power BI setting)
```

## Publishing to Power BI Service

Publish the report and configure dataset refresh.

```text
Power BI Desktop:
  File -> Publish -> Power BI Service

  Select workspace: Analytics Team

Power BI Service:
  Dataset -> Settings -> Gateway Connection

  On-Premises Data Gateway:
  - Install the gateway on a Windows machine with ODBC configured
  - Map the ODBC DSN to the gateway data source
  - Test connection

  Scheduled Refresh:
  - Frequency: Daily
  - Time:      3:00 AM, 9:00 AM, 3:00 PM (up to 8 per day)
```

## Summary

Power BI connects to ClickHouse via the ODBC driver. Use Import mode with `SummingMergeTree` tables for the fastest report performance, and use DirectQuery for real-time data with pre-aggregated views. Write M queries in Power Query to filter and transform data during import, and create DAX measures for business KPIs like profit margin and year-over-year growth. For Power BI Service scheduled refresh, install the on-premises data gateway on a Windows machine with the ODBC driver configured, and map it to the ClickHouse data source in Power BI Service settings.
