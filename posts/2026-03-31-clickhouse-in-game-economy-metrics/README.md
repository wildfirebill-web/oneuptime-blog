# How to Track In-Game Economy Metrics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, In-Game Economy, Virtual Currency, Gaming Analytics, Monetization

Description: Track and analyze in-game economy metrics in ClickHouse to understand currency flow, item transactions, and player spending patterns.

---

In-game economies are complex ecosystems with virtual currencies, item marketplaces, and real-money purchases. Monitoring inflation, sink rates, and player wealth distribution helps game designers maintain a balanced and engaging economy.

## Economy Transactions Table

```sql
CREATE TABLE economy_transactions (
    event_time          DateTime,
    player_id           UInt64,
    game_id             UInt32,
    transaction_type    LowCardinality(String),
    currency_type       LowCardinality(String),
    amount              Int64,
    item_id             UInt32,
    source              LowCardinality(String),
    date                Date DEFAULT toDate(event_time)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (game_id, currency_type, player_id, event_time);
```

## Currency Circulation: Sources vs Sinks

Track how much currency enters vs leaves the economy each day:

```sql
SELECT
    date,
    currency_type,
    sumIf(amount, transaction_type = 'earn') AS total_earned,
    abs(sumIf(amount, transaction_type = 'spend')) AS total_spent,
    sumIf(amount, transaction_type = 'earn') + sumIf(amount, transaction_type = 'spend') AS net_change
FROM economy_transactions
WHERE date >= today() - 30
GROUP BY date, currency_type
ORDER BY date, currency_type;
```

## Inflation Index

Monitor average player wealth over time as a proxy for inflation:

```sql
SELECT
    date,
    avg(balance) AS avg_player_balance,
    quantile(0.50)(balance) AS median_balance,
    quantile(0.95)(balance) AS p95_balance
FROM (
    SELECT
        date,
        player_id,
        sum(amount) AS balance
    FROM economy_transactions
    WHERE date <= today()
    GROUP BY date, player_id
)
GROUP BY date
ORDER BY date;
```

## Top Item Purchases

```sql
SELECT
    item_id,
    count() AS purchase_count,
    sum(abs(amount)) AS total_currency_spent,
    uniq(player_id) AS unique_buyers
FROM economy_transactions
WHERE transaction_type = 'spend'
  AND date >= today() - 7
GROUP BY item_id
ORDER BY purchase_count DESC
LIMIT 20;
```

## Player Spending Tiers (Whales vs Minnows)

```sql
SELECT
    multiIf(
        total_spent = 0, 'free',
        total_spent < 100, 'minnow',
        total_spent < 1000, 'dolphin',
        'whale'
    ) AS spending_tier,
    count() AS player_count,
    round(count() / sum(count()) OVER () * 100, 2) AS pct_of_players,
    sum(total_spent) AS tier_revenue
FROM (
    SELECT
        player_id,
        abs(sumIf(amount, transaction_type = 'real_money_purchase')) AS total_spent
    FROM economy_transactions
    WHERE date >= today() - 30
    GROUP BY player_id
)
GROUP BY spending_tier
ORDER BY tier_revenue DESC;
```

## Economy Health Over Time

```sql
SELECT
    toStartOfWeek(event_time) AS week,
    currency_type,
    uniq(player_id) AS active_traders,
    sum(abs(amount)) AS total_volume
FROM economy_transactions
WHERE date >= today() - 90
GROUP BY week, currency_type
ORDER BY week, currency_type;
```

## Summary

ClickHouse gives game economists a real-time view into currency flows, item demand, and player wealth distribution. By tracking sources, sinks, and net circulation daily, studios can catch runaway inflation early and make data-driven balance adjustments before they affect the player experience.
