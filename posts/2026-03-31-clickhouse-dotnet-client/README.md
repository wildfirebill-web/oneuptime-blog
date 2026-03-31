# How to Use ClickHouse .NET Client

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, .NET, C#, ClickHouse.Client, ADO.NET, Query

Description: Learn how to connect to ClickHouse from .NET using the ClickHouse.Client library, run queries, insert data, and use async patterns.

---

`ClickHouse.Client` is the most widely used .NET library for ClickHouse. It implements ADO.NET interfaces so it integrates with Dapper, Entity Framework tooling, and standard .NET data access patterns.

## Installation

```bash
dotnet add package ClickHouse.Client
```

## Connecting

```csharp
using ClickHouse.Client.ADO;

var connectionString = "Host=localhost;Port=8123;Username=default;Password=;Database=default";
using var connection = new ClickHouseConnection(connectionString);
await connection.OpenAsync();
```

For ClickHouse Cloud with TLS:

```csharp
var connectionString = "Host=abc123.clickhouse.cloud;Port=8443;Username=default;Password=secret;Protocol=https";
```

## Running a SELECT Query

```csharp
using var command = connection.CreateCommand();
command.CommandText = "SELECT number, number * number AS sq FROM numbers(5)";

using var reader = await command.ExecuteReaderAsync();
while (await reader.ReadAsync())
{
    var num = reader.GetInt64(0);
    var sq  = reader.GetInt64(1);
    Console.WriteLine($"{num} -> {sq}");
}
```

## Parameterized Queries

```csharp
command.CommandText = "SELECT * FROM events WHERE event_type = {et:String}";
command.Parameters.Add(new ClickHouseDbParameter { ParameterName = "et", Value = "pageview" });
```

## Bulk Insert with ClickHouseBulkCopy

For high-throughput inserts, use `ClickHouseBulkCopy`:

```csharp
using ClickHouse.Client.Copy;

var bulkCopy = new ClickHouseBulkCopy(connection)
{
    DestinationTableName = "events",
    BatchSize = 10_000
};

var rows = new List<object[]>
{
    new object[] { 1L, DateTime.UtcNow, "pageview" },
    new object[] { 2L, DateTime.UtcNow, "click" },
};

await bulkCopy.InitAsync();
await bulkCopy.WriteToServerAsync(rows);
```

## DDL Execution

```csharp
using var cmd = connection.CreateCommand();
cmd.CommandText = @"
    CREATE TABLE IF NOT EXISTS events (
        id         UInt64,
        event_time DateTime,
        event_type String
    ) ENGINE = MergeTree()
    ORDER BY (event_time, id)";
await cmd.ExecuteNonQueryAsync();
```

## Using Dapper

```csharp
using Dapper;

var rows = await connection.QueryAsync<dynamic>(
    "SELECT event_type, count() AS cnt FROM events GROUP BY event_type"
);
foreach (var row in rows)
    Console.WriteLine($"{row.event_type}: {row.cnt}");
```

## Summary

`ClickHouse.Client` brings standard ADO.NET patterns to ClickHouse in .NET applications. Use `ClickHouseBulkCopy` for efficient batch inserts, parameterized commands for safe queries, and Dapper for convenient object mapping. The library supports async throughout, making it suitable for ASP.NET Core and modern .NET workloads.
