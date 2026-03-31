# How to Load Dictionaries from HTTP Sources in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, HTTP Source, Data Loading, Integration

Description: Learn how to configure ClickHouse dictionaries to load their data from an HTTP or HTTPS endpoint, enabling dynamic enrichment from external APIs or services.

---

ClickHouse dictionaries can fetch their data from HTTP or HTTPS URLs. This is useful when the dimension data lives in an external system exposed via a REST API or when you want to serve dictionary data from a central object-store location.

## How HTTP Dictionary Sources Work

On each refresh cycle (defined by `LIFETIME`), ClickHouse makes an HTTP GET request to the configured URL and parses the response. The response must be in one of ClickHouse's supported formats: CSV, TSV, JSON, or others.

## Creating a Dictionary with an HTTP Source

```sql
CREATE DICTIONARY geo_country_dict
(
    country_code String,
    country_name String,
    region       String
)
PRIMARY KEY country_code
SOURCE(HTTP(
    URL 'https://data.example.com/geo/countries.csv'
    FORMAT 'CSVWithNames'
))
LAYOUT(HASHED())
LIFETIME(MIN 3600 MAX 7200);
```

- `URL` - the endpoint to fetch from
- `FORMAT` - ClickHouse format name for parsing the response body

## Adding HTTP Headers

For authenticated endpoints, add request headers:

```sql
SOURCE(HTTP(
    URL 'https://data.example.com/geo/countries.csv'
    FORMAT 'CSVWithNames'
    HEADERS(
        Authorization 'Bearer my-secret-token'
        X-App-Name    'clickhouse-dict'
    )
))
```

## Using TSV Format

```sql
SOURCE(HTTP(
    URL 'https://cdn.internal/ip_ranges.tsv'
    FORMAT 'TabSeparatedWithNames'
))
```

## Reloading on Demand

Force an immediate reload without waiting for the lifetime interval:

```sql
SYSTEM RELOAD DICTIONARY geo_country_dict;
```

## Checking the Last Load Time

```sql
SELECT name, last_successful_update_time, element_count, status
FROM system.dictionaries
WHERE name = 'geo_country_dict';
```

## Practical Example - Enriching Events with Country Names

```sql
SELECT
    event_id,
    country_code,
    dictGet('geo_country_dict', 'country_name', country_code) AS country_name,
    dictGet('geo_country_dict', 'region', country_code) AS region
FROM user_events
WHERE event_date = today()
LIMIT 20;
```

## Error Handling

If the HTTP endpoint is unavailable during a reload, ClickHouse keeps the last successfully loaded version. You can observe the failure in:

```sql
SELECT name, loading_start_time, last_exception
FROM system.dictionaries
WHERE name = 'geo_country_dict';
```

## HTTPS and SSL

ClickHouse follows HTTPS redirects and validates SSL certificates by default. Configure custom CA certificates in the ClickHouse server `config.xml` if needed.

## Summary

HTTP dictionary sources let you point ClickHouse directly at web endpoints to load enrichment data. By combining the `HTTP` source with `LIFETIME` refresh settings and optional auth headers, you can keep dictionary data fresh from external APIs or object storage without manual data loading steps.
