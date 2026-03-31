# How to Configure MySQL Connector/NET for C#

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, C#, Connector/NET, .NET, ADO.NET

Description: Set up MySQL Connector/NET in a C# project, configure connection strings, use connection pooling, and execute queries safely with parameters.

---

## Installing MySQL Connector/NET

Add the NuGet package to your project:

```bash
dotnet add package MySql.Data
```

Or for the newer, fully managed driver (recommended for new projects):

```bash
dotnet add package MySqlConnector
```

`MySqlConnector` is an open-source, high-performance alternative that is fully compatible with `MySql.Data` and supports async properly.

## Basic Connection and Query

```csharp
using MySql.Data.MySqlClient;

var connectionString = "Server=localhost;Database=mydb;User ID=app;Password=secret;";

using var connection = new MySqlConnection(connectionString);
await connection.OpenAsync();

var sql = "SELECT id, email FROM users WHERE active = @active LIMIT 10";
using var command = new MySqlCommand(sql, connection);
command.Parameters.AddWithValue("@active", 1);

using var reader = await command.ExecuteReaderAsync();
while (await reader.ReadAsync())
{
    Console.WriteLine($"ID: {reader.GetInt32("id")}, Email: {reader.GetString("email")}");
}
```

Always use parameterized queries (`@paramName`) to prevent SQL injection.

## Connection String Options

```csharp
// Full connection string with common options
var connectionString =
    "Server=localhost;" +
    "Port=3306;" +
    "Database=mydb;" +
    "User ID=app;" +
    "Password=secret;" +
    "CharSet=utf8mb4;" +           // required for emoji / full Unicode support
    "SslMode=Required;" +          // enforce TLS
    "ConnectionTimeout=10;" +      // seconds to wait for connection
    "DefaultCommandTimeout=30;";   // seconds before a query times out
```

## Configuring Connection Pooling

Connection pooling is enabled by default. Key pooling parameters:

```csharp
var connectionString =
    "Server=localhost;Database=mydb;User ID=app;Password=secret;" +
    "Pooling=true;" +
    "MinimumPoolSize=5;" +         // keep at least 5 connections open
    "MaximumPoolSize=100;" +       // hard cap on pool size
    "ConnectionLifeTime=300;";     // max seconds a connection stays in pool
```

In a high-throughput API, set `MaximumPoolSize` based on your MySQL `max_connections` divided by the number of application instances.

## Using with Microsoft.Extensions.DependencyInjection

For ASP.NET Core applications, register a connection factory:

```csharp
// Program.cs
builder.Services.AddTransient<MySqlConnection>(_ =>
    new MySqlConnection(builder.Configuration.GetConnectionString("MySql")));
```

```json
// appsettings.json
{
  "ConnectionStrings": {
    "MySql": "Server=localhost;Database=mydb;User ID=app;Password=secret;CharSet=utf8mb4;"
  }
}
```

## Executing Non-Query Commands

```csharp
using var connection = new MySqlConnection(connectionString);
await connection.OpenAsync();

// INSERT with parameters
var insertSql = "INSERT INTO orders (customer_id, total) VALUES (@customerId, @total)";
using var cmd = new MySqlCommand(insertSql, connection);
cmd.Parameters.AddWithValue("@customerId", 42);
cmd.Parameters.AddWithValue("@total", 199.99m);

int rowsAffected = await cmd.ExecuteNonQueryAsync();

// Get last inserted ID
long newOrderId = cmd.LastInsertedId;
Console.WriteLine($"Created order ID: {newOrderId}");
```

## Transactions

```csharp
using var connection = new MySqlConnection(connectionString);
await connection.OpenAsync();
using var transaction = await connection.BeginTransactionAsync();

try
{
    var deductSql = "UPDATE accounts SET balance = balance - @amount WHERE id = @from";
    using var deductCmd = new MySqlCommand(deductSql, connection, transaction);
    deductCmd.Parameters.AddWithValue("@amount", 100m);
    deductCmd.Parameters.AddWithValue("@from", 1);
    await deductCmd.ExecuteNonQueryAsync();

    var creditSql = "UPDATE accounts SET balance = balance + @amount WHERE id = @to";
    using var creditCmd = new MySqlCommand(creditSql, connection, transaction);
    creditCmd.Parameters.AddWithValue("@amount", 100m);
    creditCmd.Parameters.AddWithValue("@to", 2);
    await creditCmd.ExecuteNonQueryAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

## Handling MySQL Errors

```csharp
try
{
    await command.ExecuteNonQueryAsync();
}
catch (MySqlException ex)
{
    switch (ex.Number)
    {
        case 1062: // Duplicate entry
            Console.WriteLine("Duplicate key: " + ex.Message);
            break;
        case 1213: // Deadlock
            Console.WriteLine("Deadlock, retry the transaction");
            break;
        default:
            throw;
    }
}
```

MySQL error codes are available in `MySqlException.Number`. Common ones: 1062 (duplicate key), 1213 (deadlock), 1452 (foreign key violation).

## Summary

MySQL Connector/NET provides ADO.NET-compatible access to MySQL from C#. Use parameterized queries always, configure connection pooling via the connection string, and handle `MySqlException` by error number for duplicate keys and deadlocks. For new projects, `MySqlConnector` on NuGet is the recommended package due to better async support and active maintenance.
