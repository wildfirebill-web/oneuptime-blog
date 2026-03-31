# How to Use MySQL for Feature Flags Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Feature Flag, Schema, Deployment, Performance

Description: Learn how to implement a MySQL-backed feature flag system with user-level targeting, rollout percentages, and application-level caching for fast flag evaluation.

---

Feature flags let teams deploy code to production while controlling which users see new functionality. A MySQL-backed feature flag store is simpler to operate than a dedicated third-party service and gives you full control over targeting logic, audit history, and rollout rules.

## Feature Flag Schema

```sql
CREATE TABLE feature_flags (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  flag_key    VARCHAR(200) NOT NULL,
  description TEXT,
  is_enabled  TINYINT(1) NOT NULL DEFAULT 0,
  rollout_pct TINYINT UNSIGNED NOT NULL DEFAULT 0,
  config      JSON,
  created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  updated_by  BIGINT UNSIGNED,
  UNIQUE KEY uq_flag_key (flag_key)
) ENGINE=InnoDB;

-- Per-user overrides
CREATE TABLE feature_flag_overrides (
  flag_id    BIGINT UNSIGNED NOT NULL,
  user_id    BIGINT UNSIGNED NOT NULL,
  is_enabled TINYINT(1) NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (flag_id, user_id),
  CONSTRAINT fk_override_flag FOREIGN KEY (flag_id) REFERENCES feature_flags (id) ON DELETE CASCADE
);
```

## Creating Flags

```sql
INSERT INTO feature_flags (flag_key, description, is_enabled, rollout_pct) VALUES
  ('new_checkout',      'Redesigned checkout flow',          0, 0),
  ('ai_search',         'AI-powered product search',         1, 25),
  ('dark_mode',         'Dark mode UI option',               1, 100),
  ('beta_dashboard',    'New analytics dashboard for beta',  0, 0);
```

## Evaluating Flags in Application Code

```python
import hashlib

def is_flag_enabled(db, flag_key, user_id):
    # Check per-user override first
    override = db.execute(
        """SELECT fo.is_enabled FROM feature_flag_overrides fo
           JOIN feature_flags ff ON ff.id = fo.flag_id
           WHERE ff.flag_key = %s AND fo.user_id = %s""",
        (flag_key, user_id)
    ).fetchone()

    if override is not None:
        return bool(override[0])

    # Fall back to global flag settings
    flag = db.execute(
        "SELECT is_enabled, rollout_pct FROM feature_flags WHERE flag_key = %s",
        (flag_key,)
    ).fetchone()

    if not flag or not flag[0]:
        return False

    # Rollout: stable hash of user_id + flag_key to bucket users consistently
    if flag[1] >= 100:
        return True
    bucket = int(hashlib.md5(f"{user_id}-{flag_key}".encode()).hexdigest(), 16) % 100
    return bucket < flag[1]
```

The hash-based bucketing ensures a user is consistently in or out of a rollout rather than randomly flipping on each evaluation.

## Granting Beta Access to Specific Users

```sql
-- Enroll specific users in a flag regardless of rollout percentage
INSERT INTO feature_flag_overrides (flag_id, user_id, is_enabled)
SELECT id, 42, 1 FROM feature_flags WHERE flag_key = 'beta_dashboard'
ON DUPLICATE KEY UPDATE is_enabled = 1;
```

## Loading All Flags at Startup

Cache all flags at application startup and refresh periodically:

```python
def load_all_flags(db):
    rows = db.execute(
        "SELECT flag_key, is_enabled, rollout_pct, config FROM feature_flags"
    ).fetchall()
    return {row[0]: {"enabled": bool(row[1]), "pct": row[2], "config": row[3]}
            for row in rows}

flag_cache = load_all_flags(db)
```

Refresh the cache every 30-60 seconds in a background thread to pick up flag changes without restarting the application.

## Audit History

Track every flag change for compliance and rollback:

```sql
CREATE TABLE feature_flag_history (
  id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  flag_id    BIGINT UNSIGNED NOT NULL,
  changed_by BIGINT UNSIGNED,
  old_state  JSON,
  new_state  JSON,
  changed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## Summary

A MySQL feature flag system stores flags with rollout percentages and per-user overrides. Consistent hash-based bucketing ensures stable rollout behavior across evaluations. Application code caches all flags at startup and refreshes on a schedule, keeping evaluation latency in microseconds from memory. The system covers the core use cases - global toggles, percentage rollouts, and individual user targeting - without the cost of a third-party feature management service.
