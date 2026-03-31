# How to Use mapUpdate() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Map Function, Data Transformation, Merge

Description: Learn how mapUpdate() merges two Maps by overwriting values for existing keys in ClickHouse, with examples for config merging, overrides, and composite map building.

---

`mapUpdate()` merges two Maps by taking all entries from the second map and applying them on top of the first. For keys that exist in both maps, the value from the second map wins. For keys that exist only in the first map, they are preserved unchanged. Keys that exist only in the second map are added. This behavior mirrors how configuration overrides work: a base config combined with an environment-specific override produces a final merged configuration.

## Function Signature

```text
mapUpdate(base_map, override_map)
```

Returns a new Map. The types of both maps must match. Unlike `mapAdd()`, which sums numeric values for shared keys, `mapUpdate()` replaces them.

## Basic Usage

Observe the override behavior with a simple inline example.

```sql
SELECT mapUpdate(
    map('host', 'localhost', 'port', '5432', 'db', 'myapp'),
    map('host', 'prod-db-01', 'ssl', 'true')
) AS merged_config;
```

The result is `{'host': 'prod-db-01', 'port': '5432', 'db': 'myapp', 'ssl': 'true'}`. The `host` key from the base map is overwritten, `port` and `db` are preserved, and `ssl` is added from the override.

Contrast this with `mapAdd()` to highlight the difference.

```sql
SELECT
    mapAdd(map('clicks', 10, 'views', 50), map('clicks', 5, 'shares', 3))      AS add_result,
    mapUpdate(map('clicks', 10, 'views', 50), map('clicks', 5, 'shares', 3))   AS update_result;
```

`mapAdd` produces `{'clicks': 15, 'views': 50, 'shares': 3}` while `mapUpdate` produces `{'clicks': 5, 'views': 50, 'shares': 3}`.

## Setting Up a Sample Table

Create a table storing application configuration as Map columns, with separate base and environment-specific override maps.

```sql
CREATE TABLE app_configurations
(
    app_id       String,
    environment  String,
    base_config  Map(String, String),
    env_override Map(String, String)
)
ENGINE = MergeTree
ORDER BY (app_id, environment);

INSERT INTO app_configurations VALUES
('payments-svc', 'staging',
    map('timeout_ms', '5000', 'retries', '3', 'log_level', 'INFO', 'db_pool_size', '10'),
    map('timeout_ms', '8000', 'log_level', 'DEBUG')
),
('payments-svc', 'production',
    map('timeout_ms', '5000', 'retries', '3', 'log_level', 'INFO', 'db_pool_size', '10'),
    map('db_pool_size', '50', 'retries', '5', 'tracing', 'enabled')
),
('auth-svc', 'staging',
    map('token_ttl', '3600', 'algorithm', 'HS256', 'log_level', 'INFO'),
    map('log_level', 'DEBUG', 'mock_auth', 'true')
),
('auth-svc', 'production',
    map('token_ttl', '3600', 'algorithm', 'HS256', 'log_level', 'INFO'),
    map('token_ttl', '1800', 'algorithm', 'RS256')
);
```

## Merging Base Config with Environment Override

Use `mapUpdate()` to produce the final effective configuration for each app-environment combination.

```sql
SELECT
    app_id,
    environment,
    mapUpdate(base_config, env_override) AS effective_config
FROM app_configurations
ORDER BY app_id, environment;
```

## Extracting a Specific Merged Value

Access individual keys from the merged result using bracket notation.

```sql
SELECT
    app_id,
    environment,
    mapUpdate(base_config, env_override)['log_level']    AS effective_log_level,
    mapUpdate(base_config, env_override)['timeout_ms']   AS effective_timeout
FROM app_configurations;
```

## Applying User-Level Overrides

A common pattern in feature personalization is to start with a default settings map and apply a per-user override on top.

```sql
CREATE TABLE user_settings
(
    user_id           UInt64,
    default_settings  Map(String, String),
    user_overrides    Map(String, String)
)
ENGINE = MergeTree
ORDER BY user_id;

INSERT INTO user_settings VALUES
(101, map('theme', 'light', 'lang', 'en', 'tz', 'UTC', 'notifs', 'email'), map('theme', 'dark', 'tz', 'America/New_York')),
(102, map('theme', 'light', 'lang', 'en', 'tz', 'UTC', 'notifs', 'email'), map('lang', 'fr', 'notifs', 'none')),
(103, map('theme', 'light', 'lang', 'en', 'tz', 'UTC', 'notifs', 'email'), map());
```

Compute the final resolved settings for each user.

```sql
SELECT
    user_id,
    mapUpdate(default_settings, user_overrides) AS resolved_settings
FROM user_settings;
```

## Chaining Multiple Overrides

Apply a three-level merge by chaining `mapUpdate()` calls: defaults, then environment config, then user-specific overrides.

```sql
SELECT
    user_id,
    mapUpdate(
        mapUpdate(
            map('theme', 'light', 'lang', 'en', 'tz', 'UTC'),   -- global defaults
            map('tz', 'Europe/London', 'beta_features', 'off')   -- org-level settings
        ),
        user_overrides                                            -- user preferences
    ) AS final_settings
FROM user_settings;
```

## Summary

`mapUpdate()` implements a "right wins" merge strategy for Maps, where the second argument's values take precedence over the first for any shared keys. Use it for configuration layering, applying user preferences over defaults, merging environment-specific settings over base settings, and any scenario where a later map should override an earlier one. Unlike `mapAdd()`, it does not perform arithmetic - it replaces values. Chain multiple `mapUpdate()` calls to implement multi-level override hierarchies.
