# How to Use Date and Date32 Data Types in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Type, Date, Date32

Description: Learn the difference between Date and Date32 in ClickHouse, including storage, supported ranges, toDate() conversion, and practical table examples.

---

ClickHouse provides two date-only types for storing calendar dates without a time component: `Date` and `Date32`. The difference is in storage size and the range of dates they can represent. `Date` is compact and fast for modern data, while `Date32` extends coverage back to 1900 and forward to 2299 at the cost of twice the storage. This post explains both types, how to use them, and how to convert date values from other formats.

## Date vs Date32 at a Glance

| Type   | Underlying Storage | Min Date   | Max Date   | Storage |
|--------|--------------------|------------|------------|---------|
| Date   | UInt16 (days since epoch) | 1970-01-01 | 2149-06-06 | 2 bytes |
| Date32 | Int32 (days since epoch)  | 1900-01-01 | 2299-12-31 | 4 bytes |

`Date` stores the number of days since the Unix epoch (1970-01-01) as an unsigned 16-bit integer. This means it cannot represent dates before 1970. `Date32` uses a signed 32-bit integer, allowing dates before the epoch and far into the future.

## Creating Tables with Date Columns

```sql
CREATE TABLE user_profiles
(
    user_id      UInt64,
    username     String,
    birthdate    Date32,    -- Can be before 1970
    signup_date  Date,      -- Modern date, always >= 1970
    last_login   Date
)
ENGINE = MergeTree()
ORDER BY user_id;
```

## Inserting Date Values

Date values can be inserted as string literals in `YYYY-MM-DD` format.

```sql
INSERT INTO user_profiles
    (user_id, username, birthdate, signup_date, last_login) VALUES
(1, 'alice',   '1985-04-12', '2022-06-01', '2026-03-30'),
(2, 'bob',     '1942-11-30', '2020-01-15', '2026-03-28'),
(3, 'charlie', '2000-07-04', '2023-09-10', '2026-03-31');
```

## Converting to Date with toDate()

Use `toDate()` to convert strings, Unix timestamps, or DateTime values to a Date.

```sql
SELECT
    toDate('2026-03-31')           AS from_string,
    toDate(1743379200)             AS from_unix_timestamp,
    toDate(now())                  AS from_datetime,
    toDate32('1950-06-15')         AS date32_from_string,
    toDate32(-7305)                AS date32_from_int;   -- Days before epoch are negative
```

## Date Arithmetic

ClickHouse supports arithmetic operations on Date values. Adding or subtracting an integer changes the date by that many days.

```sql
SELECT
    signup_date,
    signup_date + 30              AS plus_30_days,
    signup_date - 7               AS minus_7_days,
    last_login - signup_date      AS days_since_signup,
    dateDiff('day', signup_date, last_login) AS diff_days
FROM user_profiles;
```

## Extracting Date Components

```sql
SELECT
    signup_date,
    toYear(signup_date)    AS year,
    toMonth(signup_date)   AS month,
    toDayOfMonth(signup_date) AS day,
    toDayOfWeek(signup_date)  AS day_of_week,  -- 1=Monday, 7=Sunday
    toWeek(signup_date)    AS week_of_year,
    toQuarter(signup_date) AS quarter
FROM user_profiles;
```

## Date Truncation

Use `toStartOfMonth`, `toStartOfWeek`, and similar functions to group date data by period.

```sql
SELECT
    toStartOfMonth(signup_date)  AS signup_month,
    toStartOfWeek(last_login)    AS last_login_week,
    count()                       AS user_count
FROM user_profiles
GROUP BY signup_month, last_login_week
ORDER BY signup_month;
```

## Filtering by Date Range

```sql
-- Users who signed up in 2022
SELECT user_id, username, signup_date
FROM user_profiles
WHERE signup_date >= toDate('2022-01-01')
  AND signup_date < toDate('2023-01-01');

-- Users born before 1970 (requires Date32)
SELECT user_id, username, birthdate
FROM user_profiles
WHERE birthdate < toDate32('1970-01-01');
```

## Comparing Date and Date32

Mixing Date and Date32 in expressions requires explicit casting to avoid type errors.

```sql
SELECT
    user_id,
    birthdate,
    signup_date,
    dateDiff('year', toDate32(signup_date), toDate32(last_login)) AS years_active
FROM user_profiles;
```

## When to Use Date vs Date32

Use `Date` when:
- All dates are guaranteed to be in 1970-2149 range
- Storage efficiency matters and you have many rows
- You are storing event dates, transaction dates, or log dates

Use `Date32` when:
- Data includes historical dates before 1970 (birthdates, historical records)
- You need dates beyond 2149 (long-range forecasting, expiry dates)

## Summary

`Date` and `Date32` are ClickHouse's date-only types, storing calendar dates without time or timezone information. `Date` uses 2 bytes and covers 1970-2149, while `Date32` uses 4 bytes and extends from 1900 to 2299. For modern application data, `Date` is the compact default choice. Use `Date32` when your data spans historical dates before 1970 or requires a wider future range. Both support arithmetic operations, date extraction functions, and integration with `DateTime` through `toDate()` conversion.
