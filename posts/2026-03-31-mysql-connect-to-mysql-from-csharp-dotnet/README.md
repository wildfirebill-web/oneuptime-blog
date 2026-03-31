# How to Connect to MySQL from C#/.NET

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CSharp, DotNet, Connection, Driver

Description: Learn how to connect to a MySQL database from C# and .NET using MySqlConnector or Connector/NET with ADO.NET and Dapper examples.

---

## Overview

.NET applications connect to MySQL through ADO.NET providers. The two main options are:

| Driver | Package | Notes |
|--------|---------|-------|
| MySqlConnector | `MySqlConnector` | Community-maintained, async-first, recommended |
| Oracle Connector/NET | `MySql.Data` | Official Oracle driver |

This guide uses `MySqlConnector` as it has better async support and is more actively maintained.

## Installing MySqlConnector

```bash
dotnet add package MySqlConnector
```

For Entity Framework Core:

```bash
dotnet add package Pomelo.EntityFrameworkCore.MySql
```

## Basic Connection

```csharp
using MySqlConnector;

string connectionString =
    "Server=localhost;Port=3306;Database=shop;User=app_user;Password=secret;" +
    "CharSet=utf8mb4;SslMode=None;";

await using var connection = new MySqlConnection(connectionString);
await connection.OpenAsync();

Console.WriteLine($"Connected to MySQL {connection.ServerVersion}");
```

## Querying Data

```csharp
string sql = "SELECT id, name, price FROM products WHERE price < @maxPrice";

await using var cmd = new MySqlCommand(sql, connection);
cmd.Parameters.AddWithValue("@maxPrice", 50.00);

await using var reader = await cmd.ExecuteReaderAsync();
while (await reader.ReadAsync())
{
    Console.WriteLine($"{reader.GetInt32("id")}  {reader.GetString("name")}  {reader.GetDouble("price")}");
}
```

## Inserting Data

```csharp
string insertSql = "INSERT INTO orders (customer_id, total) VALUES (@customerId, @total)";

await using var insertCmd = new MySqlCommand(insertSql, connection);
insertCmd.Parameters.AddWithValue("@customerId", 42);
insertCmd.Parameters.AddWithValue("@total", 99.99);

await insertCmd.ExecuteNonQueryAsync();
long newId = insertCmd.LastInsertedId;
Console.WriteLine($"Inserted order ID: {newId}");
```

## Using Dapper

Dapper is a lightweight micro-ORM that simplifies result mapping:

```bash
dotnet add package Dapper
```

```csharp
using Dapper;

var products = await connection.QueryAsync<Product>(
    "SELECT id, name, price FROM products WHERE price < @MaxPrice",
    new { MaxPrice = 50.00 }
);

foreach (var product in products)
    Console.WriteLine($"{product.Name}: ${product.Price}");

public record Product(int Id, string Name, double Price);
```

## Transactions

```csharp
await using var transaction = await connection.BeginTransactionAsync();
try
{
    var cmd1 = new MySqlCommand("UPDATE stock SET qty = qty - 1 WHERE id = 7",
                                connection, transaction);
    await cmd1.ExecuteNonQueryAsync();

    var cmd2 = new MySqlCommand("INSERT INTO order_items (item_id) VALUES (7)",
                                connection, transaction);
    await cmd2.ExecuteNonQueryAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

## Connection Pooling

`MySqlConnector` handles pooling transparently. Configure via the connection string:

```text
Server=localhost;Database=shop;User=app_user;Password=secret;
MinimumPoolSize=2;MaximumPoolSize=20;ConnectionTimeout=30;
```

## Summary

Use `MySqlConnector` for modern async-first MySQL connectivity in .NET. Always use named parameters (`@paramName`) to prevent SQL injection, configure pooling in the connection string, and set `CharSet=utf8mb4`. Dapper simplifies result mapping with minimal overhead compared to full ORMs.
