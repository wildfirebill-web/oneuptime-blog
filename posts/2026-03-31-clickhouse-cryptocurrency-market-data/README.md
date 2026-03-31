# How to Track Cryptocurrency Market Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cryptocurrency, Market Data, OHLCV, Time Series, Finance

Description: Store and analyze cryptocurrency market data in ClickHouse - OHLCV candles, order book snapshots, trade feeds, and cross-exchange price analytics.

---

Cryptocurrency markets generate enormous volumes of tick data - trades, order book updates, and funding rates across hundreds of pairs and dozens of exchanges. ClickHouse is purpose-built for this kind of high-frequency time-series workload.

## Trade Tick Table

```sql
CREATE TABLE crypto_trades (
    exchange     LowCardinality(String),
    symbol       LowCardinality(String),
    trade_id     String,
    side         LowCardinality(String),  -- buy, sell
    price        Float64,
    quantity     Float64,
    quote_qty    Float64,
    traded_at    DateTime64(3)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(traded_at)
ORDER BY (exchange, symbol, traded_at);
```

## OHLCV Candle Computation

Build 1-minute candles from raw trades:

```sql
SELECT
    exchange,
    symbol,
    toStartOfMinute(traded_at) AS candle_open,
    argMin(price, traded_at) AS open,
    max(price) AS high,
    min(price) AS low,
    argMax(price, traded_at) AS close,
    sum(quantity) AS volume,
    sum(quote_qty) AS quote_volume,
    count() AS trade_count
FROM crypto_trades
WHERE traded_at >= now() - INTERVAL 1 HOUR
  AND symbol = 'BTCUSDT'
GROUP BY exchange, symbol, candle_open
ORDER BY candle_open;
```

## Volume-Weighted Average Price (VWAP)

```sql
SELECT
    symbol,
    toStartOfHour(traded_at) AS hour,
    sum(price * quantity) / sum(quantity) AS vwap,
    sum(quantity) AS total_volume,
    countIf(side = 'buy') AS buy_trades,
    countIf(side = 'sell') AS sell_trades
FROM crypto_trades
WHERE traded_at >= today()
GROUP BY symbol, hour
ORDER BY symbol, hour;
```

## Cross-Exchange Price Divergence

Identify arbitrage opportunities between exchanges:

```sql
SELECT
    symbol,
    toStartOfMinute(traded_at) AS minute,
    maxIf(price, exchange = 'Binance') AS binance_price,
    maxIf(price, exchange = 'Coinbase') AS coinbase_price,
    abs(maxIf(price, exchange = 'Binance') - maxIf(price, exchange = 'Coinbase')) AS price_diff,
    round(abs(maxIf(price, exchange = 'Binance') - maxIf(price, exchange = 'Coinbase'))
        / minIf(price, exchange = 'Binance') * 100, 4) AS diff_pct
FROM crypto_trades
WHERE traded_at >= now() - INTERVAL 1 HOUR
  AND symbol = 'BTCUSDT'
  AND exchange IN ('Binance', 'Coinbase')
GROUP BY symbol, minute
HAVING diff_pct > 0.1
ORDER BY minute;
```

## Trade Size Distribution

Understand the profile of buyers and sellers:

```sql
SELECT
    symbol,
    side,
    multiIf(
        quantity < 0.01, 'micro',
        quantity < 0.1,  'small',
        quantity < 1.0,  'medium',
        quantity < 10.0, 'large',
        'whale'
    ) AS size_bucket,
    count() AS trade_count,
    sum(quote_qty) AS usd_volume
FROM crypto_trades
WHERE traded_at >= today()
  AND symbol = 'BTCUSDT'
GROUP BY symbol, side, size_bucket
ORDER BY side, usd_volume DESC;
```

## 24-Hour Market Summary

```sql
SELECT
    symbol,
    argMin(price, traded_at) AS open_24h,
    argMax(price, traded_at) AS close_24h,
    max(price) AS high_24h,
    min(price) AS low_24h,
    round((argMax(price, traded_at) - argMin(price, traded_at)) / argMin(price, traded_at) * 100, 2) AS change_pct_24h,
    sum(quote_qty) AS volume_usd_24h
FROM crypto_trades
WHERE traded_at >= now() - INTERVAL 24 HOUR
GROUP BY symbol
ORDER BY volume_usd_24h DESC;
```

## Summary

ClickHouse is a natural fit for cryptocurrency market data - it can ingest millions of trade ticks per second, compute OHLCV candles on the fly, calculate VWAP, and detect cross-exchange price divergences with millisecond-level timestamps. Partitioning by day keeps storage efficient for high-frequency data retention.
