# How to Use MySQL for Configuration Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Configuration, Cache, Schema, Best Practice

Description: Learn how to store and manage application configuration in MySQL with versioning, environment scoping, caching patterns, and change history for auditable runtime settings.

---

Storing application configuration in MySQL allows runtime changes without redeployment, supports environment-specific overrides, and provides a queryable audit trail of every settings change. It works well alongside environment variables: secrets stay in env vars, while feature switches and tunable parameters live in the database.

## Configuration Schema

```sql
CREATE TABLE config_items (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  environment VARCHAR(50) NOT NULL DEFAULT 'production',
  namespace   VARCHAR(100) NOT NULL,
  key_name    VARCHAR(200) NOT NULL,
  value       TEXT NOT NULL,
  value_type  ENUM('string', 'integer', 'float', 'boolean', 'json') NOT NULL DEFAULT 'string',
  description TEXT,
  is_active   TINYINT(1) NOT NULL DEFAULT 1,
  updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  updated_by  BIGINT UNSIGNED,
  UNIQUE KEY uq_env_ns_key (environment, namespace, key_name),
  INDEX idx_namespace (environment, namespace)
) ENGINE=InnoDB;

-- Audit history of every change
CREATE TABLE config_history (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  config_id   BIGINT UNSIGNED NOT NULL,
  old_value   TEXT,
  new_value   TEXT,
  changed_by  BIGINT UNSIGNED,
  changed_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_config_id (config_id)
);
```

## Inserting Configuration Values

```sql
INSERT INTO config_items (environment, namespace, key_name, value, value_type, description) VALUES
  ('production', 'email',   'max_daily_sends',  '10000',    'integer', 'Max emails per day per tenant'),
  ('production', 'search',  'results_per_page', '24',       'integer', 'Default search results per page'),
  ('production', 'payment', 'retry_attempts',   '3',        'integer', 'Payment retry count on failure'),
  ('staging',    'email',   'max_daily_sends',  '100',      'integer', 'Lower limit for staging'),
  ('production', 'feature', 'new_checkout',     'false',    'boolean', 'Enable new checkout flow');
```

## Reading Configuration in Application Code

```python
import json
from functools import lru_cache

@lru_cache(maxsize=256)
def get_config(db, environment, namespace, key, default=None):
    row = db.execute(
        "SELECT value, value_type FROM config_items "
        "WHERE environment = %s AND namespace = %s AND key_name = %s AND is_active = 1",
        (environment, namespace, key)
    ).fetchone()

    if not row:
        return default

    value, value_type = row
    if value_type == 'integer':
        return int(value)
    elif value_type == 'float':
        return float(value)
    elif value_type == 'boolean':
        return value.lower() in ('true', '1', 'yes')
    elif value_type == 'json':
        return json.loads(value)
    return value

# Usage
max_sends = get_config(db, 'production', 'email', 'max_daily_sends', default=1000)
```

## Updating Configuration with History

Record the previous value before updating:

```sql
START TRANSACTION;

INSERT INTO config_history (config_id, old_value, new_value, changed_by)
SELECT id, value, '50000', 1
FROM config_items
WHERE environment = 'production' AND namespace = 'email' AND key_name = 'max_daily_sends';

UPDATE config_items
SET value = '50000', updated_by = 1
WHERE environment = 'production' AND namespace = 'email' AND key_name = 'max_daily_sends';

COMMIT;
```

## Loading All Config for a Namespace

Load an entire namespace at startup to minimize database roundtrips:

```sql
SELECT key_name, value, value_type
FROM config_items
WHERE environment = 'production'
  AND namespace = 'email'
  AND is_active = 1;
```

Cache the result in application memory and refresh it on a schedule (every 60 seconds) rather than querying on every request.

## Summary

MySQL configuration management stores typed key-value pairs scoped by environment and namespace, with a history table capturing every change for audit purposes. Application code reads config at startup into a local cache and refreshes periodically, keeping database load minimal. The `value_type` column enables automatic casting to the correct native type in the application layer.
