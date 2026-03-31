# How to Use geohashEncode() and geohashDecode() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Geospatial, Geohash, Analytics, Location

Description: Learn how geohashEncode() and geohashDecode() convert between geographic coordinates and geohash strings in ClickHouse for spatial indexing and proximity queries.

---

Geohash is a hierarchical spatial indexing system that encodes a (latitude, longitude) pair as a short alphanumeric string. Nearby locations share a common prefix, making geohashes useful for proximity grouping, spatial indexing, and tile-based analytics. ClickHouse provides `geohashEncode(lon, lat, precision)` to encode coordinates into a geohash string and `geohashDecode(hash)` to decode a geohash back to its center (longitude, latitude). Precision ranges from 1 (very coarse, ~5000 km) to 12 (very fine, ~3.7 cm).

## Basic Usage

```sql
-- Encode a location to different precision levels
SELECT
    geohashEncode(-122.4194, 37.7749, 4)  AS geohash_4,   -- ~39 km x 20 km
    geohashEncode(-122.4194, 37.7749, 6)  AS geohash_6,   -- ~1.2 km x 0.6 km
    geohashEncode(-122.4194, 37.7749, 8)  AS geohash_8,   -- ~38 m x 19 m
    geohashEncode(-122.4194, 37.7749, 12) AS geohash_12;  -- ~3.7 cm x 1.9 cm
```

```text
geohash_4  geohash_6  geohash_8  geohash_12
9q8y       9q8yy9     9q8yy9km   9q8yy9kmu...
```

## Decoding a Geohash

`geohashDecode()` returns a tuple `(longitude, latitude)` of the cell center.

```sql
-- Decode geohashes back to coordinates
SELECT
    hash,
    geohashDecode(hash).1 AS center_lon,
    geohashDecode(hash).2 AS center_lat
FROM (
    SELECT arrayJoin(['9q8y', '9q8yy9', '9q8yy9km']) AS hash
);
```

```text
hash      center_lon   center_lat
9q8y      -122.3291    37.7490
9q8yy9    -122.4194    37.7744
9q8yy9km  -122.4194    37.7749
```

## Grouping Events by Geohash Cell

```sql
-- Aggregate events by geohash cell (precision 6 ~ neighborhood level)
SELECT
    geohashEncode(longitude, latitude, 6) AS cell,
    count()                               AS event_count,
    uniq(user_id)                         AS unique_users,
    avg(duration_ms)                      AS avg_duration
FROM location_events
WHERE event_time >= today() - 7
GROUP BY cell
ORDER BY event_count DESC
LIMIT 20;
```

## Proximity Search Using Geohash Prefix

Two locations sharing the same geohash prefix are guaranteed to be nearby. Use prefix matching for fast approximate proximity queries without computing distances.

```sql
-- Find events in the same geohash-6 cell as a reference point
WITH geohashEncode(-122.4194, 37.7749, 6) AS ref_cell
SELECT
    event_id,
    longitude,
    latitude,
    geohashEncode(longitude, latitude, 6) AS cell
FROM location_events
WHERE geohashEncode(longitude, latitude, 6) = ref_cell
LIMIT 20;
```

## Storing Geohash as a Materialized Column

Precompute the geohash at insert time to avoid per-query recomputation.

```sql
CREATE TABLE location_events_geo (
    event_id    UUID DEFAULT generateUUIDv4(),
    event_time  DateTime,
    longitude   Float64,
    latitude    Float64,
    geohash6    String MATERIALIZED geohashEncode(longitude, latitude, 6),
    geohash8    String MATERIALIZED geohashEncode(longitude, latitude, 8)
) ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (geohash6, event_time);
```

Ordering by `geohash6` groups spatially nearby rows together, improving compression and range scan performance for cell-level queries.

## Building a Heatmap Tile Aggregation

```sql
-- Aggregate counts for heatmap rendering at zoom level 6 (~1 km cells)
SELECT
    geohashEncode(longitude, latitude, 6)  AS tile,
    geohashDecode(geohashEncode(longitude, latitude, 6)).1 AS tile_lon,
    geohashDecode(geohashEncode(longitude, latitude, 6)).2 AS tile_lat,
    count()                                AS events
FROM location_events
WHERE event_time >= today() - 1
GROUP BY tile
ORDER BY events DESC
LIMIT 100;
```

## Detecting Geohash Neighbors for Proximity

Adjacent geohash cells do not always share a prefix. Use `geohashesInBox()` to get all cells in a bounding area.

```sql
-- Get the set of geohash-6 cells covering a 0.1-degree bounding box
SELECT arrayJoin(geohashesInBox(
    -122.47, 37.75,    -- min lon, min lat
    -122.38, 37.81,    -- max lon, max lat
    6                  -- precision
)) AS cell;
```

## Geohash Hierarchy: Coarser Cells from a Fine Hash

A geohash of precision N is the prefix of a geohash of precision N+1. Use `substring()` to navigate up the hierarchy.

```sql
-- Derive coarser cells from a precision-8 geohash
SELECT
    geohash8,
    substring(geohash8, 1, 6) AS geohash6,
    substring(geohash8, 1, 4) AS geohash4,
    substring(geohash8, 1, 2) AS geohash2
FROM (
    SELECT geohashEncode(-122.4194, 37.7749, 8) AS geohash8
);
```

## Summary

`geohashEncode(lon, lat, precision)` encodes a geographic coordinate to a geohash string of length 1 to 12. `geohashDecode(hash)` returns the cell center as a `(longitude, latitude)` tuple. Use geohashes as materialized columns ordered in the sort key to co-locate spatially nearby rows, as grouping keys for heatmap aggregations, and as approximate proximity tokens via prefix matching. For coverage queries over a bounding box, `geohashesInBox()` produces all cells at a given precision that intersect the box.
