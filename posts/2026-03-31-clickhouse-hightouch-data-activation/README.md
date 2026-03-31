# How to Use ClickHouse with Hightouch for Data Activation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Hightouch, Data Activation, Reverse ETL, Personalization

Description: Use Hightouch with ClickHouse to activate your warehouse data by syncing user traits, segments, and events into marketing and CRM tools automatically.

---

## What Is Data Activation

Data activation is the practice of making warehouse data operational - pushing computed insights back into the tools your marketing, sales, and product teams use daily. Hightouch is a leading reverse ETL platform that reads from your warehouse (including ClickHouse) and writes to 200+ destinations.

## Connecting ClickHouse to Hightouch

In Hightouch, add a ClickHouse source:

1. Navigate to Sources and add a new source
2. Select ClickHouse
3. Enter your connection details and test the connection

Create a dedicated read-only user:

```sql
CREATE USER hightouch_reader
    IDENTIFIED WITH sha256_password BY 'strong_password'
    HOST IP '34.0.0.0/8';  -- Hightouch IP range

CREATE ROLE hightouch_read;
GRANT SELECT ON analytics.* TO hightouch_read;
GRANT hightouch_read TO hightouch_reader;
```

## Defining a Model in Hightouch

Hightouch reads SQL models. Define a user profile model for personalization:

```sql
SELECT
    user_id,
    email,
    first_name,
    last_name,
    plan_type,
    usage_score,
    days_since_signup,
    feature_flags,
    preferred_language,
    last_active_at
FROM analytics.user_profiles_current
WHERE is_active = 1
```

## Syncing to Braze for Personalization

Map model fields to Braze user attributes:

```text
Source model: user_profiles
Destination: Braze Users
Match on: user_id -> external_id
Fields:
  plan_type -> plan_type (string attribute)
  usage_score -> usage_score (number attribute)
  preferred_language -> language
Schedule: Every 1 hour
```

Braze now has fresh usage scores and plan data for personalizing in-app messages and emails without requiring backend API calls per request.

## Behavioral Segments for Ad Targeting

Compute a churn-risk segment and push to Google Ads:

```sql
SELECT
    u.email,
    u.user_id
FROM analytics.users u
JOIN analytics.usage_weekly uw ON uw.user_id = u.user_id
WHERE uw.week_start = toMonday(today())
  AND uw.sessions < 2
  AND u.plan_type = 'paid'
  AND u.days_since_signup > 14
```

Sync this segment to a Google Ads Customer Match list for win-back campaigns.

## Incremental Syncs for Performance

Configure Hightouch to only sync changed records using a cursor column:

```text
Sync type: Incremental
Cursor column: updated_at
Primary key: user_id
```

Hightouch tracks the high watermark and only reads rows where `updated_at` is newer than the last sync, keeping ClickHouse load low.

## Tracking Activation Metrics

Measure the business impact of data activation by tracking conversion rates per segment in ClickHouse:

```sql
SELECT
    segment_name,
    count(DISTINCT user_id) AS segment_size,
    countIf(converted = 1) AS conversions,
    round(countIf(converted = 1) / count(DISTINCT user_id) * 100, 2) AS conversion_rate
FROM analytics.activation_experiments
GROUP BY segment_name
ORDER BY conversion_rate DESC;
```

## Summary

Hightouch with ClickHouse enables data activation by transforming warehouse insights into operational actions - syncing user traits, behavioral segments, and computed scores into marketing and CRM tools to drive personalization and conversion without custom integrations.
