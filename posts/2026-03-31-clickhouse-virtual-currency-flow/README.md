# How to Track Virtual Currency Flow in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Gaming Analytics, Virtual Currency, Economy, Transaction

Description: Learn how to model and query virtual currency transactions in ClickHouse to monitor game economy health, detect inflation, and audit player balances.

---

Virtual economies in live service games can generate millions of currency transactions per day. Tracking coin mints, item purchases, and transfers accurately is essential for balancing the game economy and detecting fraud. ClickHouse handles this volume with ease.

## Schema Design for Currency Transactions

```sql
CREATE TABLE currency_transactions (
    tx_id           UUID,
    player_id       UInt64,
    game_id         UInt32,
    occurred_at     DateTime,
    currency_type   LowCardinality(String),  -- 'gold', 'gems', 'credits'
    tx_type         LowCardinality(String),  -- 'earn', 'spend', 'purchase', 'transfer'
    amount          Int64,                   -- positive = credit, negative = debit
    balance_after   Int64,
    source          LowCardinality(String),  -- 'quest_reward', 'shop', 'iap'
    item_id         UInt32
) ENGINE = MergeTree()
ORDER BY (game_id, player_id, occurred_at)
PARTITION BY toYYYYMM(occurred_at);
```

Using signed `Int64` for `amount` allows a single table to represent both credits and debits.

## Daily Currency Minted vs Spent

```sql
SELECT
    toDate(occurred_at)          AS day,
    currency_type,
    sumIf(amount, amount > 0)    AS total_earned,
    sumIf(-amount, amount < 0)   AS total_spent,
    sumIf(amount, amount > 0) + sumIf(amount, amount < 0) AS net_flow
FROM currency_transactions
WHERE game_id = 42
  AND occurred_at >= today() - 30
GROUP BY day, currency_type
ORDER BY day, currency_type;
```

A positive `net_flow` means inflation - more currency entering the economy than leaving.

## Top Currency Sinks (Where Players Spend)

```sql
SELECT
    source,
    count()          AS transactions,
    sum(-amount)     AS total_spent
FROM currency_transactions
WHERE tx_type = 'spend'
  AND game_id = 42
  AND occurred_at >= today() - 7
GROUP BY source
ORDER BY total_spent DESC
LIMIT 10;
```

## Player Balance Distribution

```sql
SELECT
    multiIf(
        balance_after < 100,   'broke',
        balance_after < 1000,  'low',
        balance_after < 10000, 'medium',
        'whale'
    ) AS balance_tier,
    uniqExact(player_id) AS player_count
FROM (
    SELECT
        player_id,
        argMax(balance_after, occurred_at) AS balance_after
    FROM currency_transactions
    WHERE game_id = 42
      AND currency_type = 'gold'
    GROUP BY player_id
)
GROUP BY balance_tier;
```

## Suspicious Accumulation Detection

Flag players who earned an unusually high amount in a single day - a signal of exploits or bot activity.

```sql
SELECT
    player_id,
    toDate(occurred_at) AS day,
    sum(amount)         AS daily_earned
FROM currency_transactions
WHERE tx_type = 'earn'
  AND game_id = 42
  AND occurred_at >= today() - 7
GROUP BY player_id, day
HAVING daily_earned > (
    SELECT quantile(0.999)(daily_sum)
    FROM (
        SELECT player_id, toDate(occurred_at) AS d, sum(amount) AS daily_sum
        FROM currency_transactions
        WHERE tx_type = 'earn' AND game_id = 42
        GROUP BY player_id, d
    )
)
ORDER BY daily_earned DESC;
```

## Materialized View for Economy Snapshots

```sql
CREATE MATERIALIZED VIEW currency_daily_mv
ENGINE = SummingMergeTree()
ORDER BY (game_id, currency_type, day)
AS
SELECT
    game_id,
    currency_type,
    toDate(occurred_at)         AS day,
    sumIf(amount, amount > 0)   AS earned,
    sumIf(-amount, amount < 0)  AS spent
FROM currency_transactions
GROUP BY game_id, currency_type, day;
```

## Summary

Modeling virtual currency as a signed transaction ledger in ClickHouse gives you a complete audit trail and enables powerful economy health queries. Materialized views keep dashboards fast, while quantile-based anomaly detection surfaces potential exploits before they destabilize the game economy.
