# How to Use ClickHouse with Kotlin

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Kotlin, JDBC, Coroutines, Analytics

Description: Connect to ClickHouse from Kotlin using the JDBC driver with coroutines for async query execution in JVM analytics applications.

---

## Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.clickhouse:clickhouse-jdbc:0.6.0")
    implementation("com.zaxxer:HikariCP:5.1.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
}
```

## DataSource Setup

```kotlin
import com.zaxxer.hikari.HikariConfig
import com.zaxxer.hikari.HikariDataSource

fun buildDataSource(): HikariDataSource {
    val config = HikariConfig().apply {
        jdbcUrl          = "jdbc:ch://localhost:8123/analytics"
        username         = "default"
        password         = ""
        driverClassName  = "com.clickhouse.jdbc.ClickHouseDriver"
        maximumPoolSize  = 10
        minimumIdle      = 2
    }
    return HikariDataSource(config)
}

val dataSource = buildDataSource()
```

## Data Class and Query Function

```kotlin
data class EventSummary(
    val eventName: String,
    val count:     Long,
    val avgMs:     Double
)

fun getTopEvents(days: Int): List<EventSummary> {
    val sql = """
        SELECT event_name, count() AS cnt, avg(duration_ms) AS avg_ms
        FROM events
        WHERE ts >= now() - INTERVAL ? DAY
        GROUP BY event_name
        ORDER BY cnt DESC
        LIMIT 20
    """.trimIndent()

    return dataSource.connection.use { conn ->
        conn.prepareStatement(sql).use { ps ->
            ps.setInt(1, days)
            ps.executeQuery().use { rs ->
                buildList {
                    while (rs.next()) {
                        add(EventSummary(
                            eventName = rs.getString("event_name"),
                            count     = rs.getLong("cnt"),
                            avgMs     = rs.getDouble("avg_ms")
                        ))
                    }
                }
            }
        }
    }
}
```

## Coroutines with Dispatchers.IO

Wrap blocking JDBC calls in `withContext(Dispatchers.IO)` for coroutine-safe execution.

```kotlin
import kotlinx.coroutines.*

suspend fun getTopEventsAsync(days: Int): List<EventSummary> =
    withContext(Dispatchers.IO) {
        getTopEvents(days)
    }

fun main() = runBlocking {
    val results = getTopEventsAsync(7)
    results.forEach { println("${it.eventName}: ${it.count}") }
}
```

## Batch Insert

```kotlin
fun insertEvents(events: List<Event>) {
    val sql = "INSERT INTO events (user_id, event_name, ts) VALUES (?, ?, ?)"
    dataSource.connection.use { conn ->
        conn.prepareStatement(sql).use { ps ->
            events.chunked(5_000).forEach { batch ->
                batch.forEach { e ->
                    ps.setLong(1, e.userId)
                    ps.setString(2, e.name)
                    ps.setTimestamp(3, java.sql.Timestamp.from(e.ts))
                    ps.addBatch()
                }
                ps.executeBatch()
                ps.clearBatch()
            }
        }
    }
}
```

## Ktor Integration

```kotlin
routing {
    get("/api/events") {
        val days = call.request.queryParameters["days"]?.toIntOrNull() ?: 7
        val data = getTopEventsAsync(days)
        call.respond(data)
    }
}
```

## Summary

Kotlin uses the ClickHouse JDBC driver wrapped in HikariCP for connection pooling. Use `withContext(Dispatchers.IO)` to bridge blocking JDBC calls into coroutine-based services, and `chunked` for safe batch inserts without overwhelming the driver.
