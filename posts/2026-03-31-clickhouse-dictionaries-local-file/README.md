# How to Load Dictionaries from Local Files in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, File Source, CSV, Data Loading

Description: Learn how to load ClickHouse dictionaries from local CSV, TSV, and other file formats stored on the ClickHouse server for fast in-memory enrichment.

---

Local file sources are the simplest way to load a ClickHouse dictionary. You place a flat file (CSV, TSV, or other supported format) on the ClickHouse server and point the dictionary at it. On each reload, ClickHouse reads the file and builds the in-memory lookup structure.

## Preparing the File

Create a CSV file on the ClickHouse server:

```bash
cat /var/lib/clickhouse/user_files/countries.csv
```

```text
country_code,country_name,region,continent
US,United States,Americas,North America
DE,Germany,Europe,Europe
JP,Japan,Asia,Asia
BR,Brazil,Americas,South America
```

## Creating the Dictionary

```sql
CREATE DICTIONARY country_dict
(
    country_code String,
    country_name String,
    region       String,
    continent    String
)
PRIMARY KEY country_code
SOURCE(FILE(
    PATH   '/var/lib/clickhouse/user_files/countries.csv'
    FORMAT 'CSVWithNames'
))
LAYOUT(HASHED())
LIFETIME(MIN 600 MAX 1200);
```

ClickHouse re-reads the file every time the `LIFETIME` interval triggers a reload.

## Using TSV Format

```sql
SOURCE(FILE(
    PATH   '/var/lib/clickhouse/user_files/countries.tsv'
    FORMAT 'TabSeparatedWithNames'
))
```

## File Permissions

The file must be readable by the `clickhouse` OS user. ClickHouse restricts file sources to paths under the `user_files_path` setting (default: `/var/lib/clickhouse/user_files/`).

## Integer Key Example

```sql
CREATE DICTIONARY status_code_dict
(
    code        UInt16,
    description String
)
PRIMARY KEY code
SOURCE(FILE(
    PATH   '/var/lib/clickhouse/user_files/status_codes.csv'
    FORMAT 'CSVWithNames'
))
LAYOUT(FLAT())
LIFETIME(MIN 0 MAX 0);
```

`LIFETIME(MIN 0 MAX 0)` means the dictionary is loaded once at startup and never refreshed automatically.

## Reloading After File Update

```sql
SYSTEM RELOAD DICTIONARY country_dict;
```

## Querying the Dictionary

```sql
SELECT
    user_id,
    country_code,
    dictGet('country_dict', 'country_name', country_code) AS country_name,
    dictGet('country_dict', 'continent',    country_code) AS continent
FROM registrations
WHERE event_date = today();
```

## Checking Status

```sql
SELECT name, element_count, loading_duration, status
FROM system.dictionaries
WHERE name = 'country_dict';
```

## Updating the Dictionary Data

To update the data, replace the file and reload:

```bash
cp /tmp/countries_new.csv /var/lib/clickhouse/user_files/countries.csv
```

```sql
SYSTEM RELOAD DICTIONARY country_dict;
```

## Summary

File-based dictionary sources are the fastest way to bootstrap a ClickHouse dictionary when data is small and relatively static. By placing a CSV or TSV file in the allowed user_files path and specifying the `FILE` source, you get an in-memory lookup structure that reloads automatically based on the configured lifetime.
