# How to Use Change Streams with the MongoDB .NET Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, CSharp, Change Stream, DotNet, Real-Time

Description: Learn how to watch real-time document changes in MongoDB collections using Change Streams with the .NET Driver in C#.

---

## What Are Change Streams?

Change Streams provide a real-time stream of change events (inserts, updates, replaces, deletes) from a MongoDB collection, database, or deployment. They are built on the oplog and are resumable - your application can restart from the last processed event using a resume token.

## Prerequisites

- MongoDB 3.6+ replica set or sharded cluster (Change Streams require an oplog).

```bash
dotnet add package MongoDB.Driver
```

## Watching a Collection

```csharp
using MongoDB.Driver;
using System;
using System.Threading;
using System.Threading.Tasks;

var client = new MongoClient("mongodb://localhost:27017");
var db = client.GetDatabase("shopdb");
var products = db.GetCollection<Product>("products");

using var cursor = await products.WatchAsync();

await cursor.ForEachAsync(change =>
{
    Console.WriteLine($"Operation: {change.OperationType}");
    Console.WriteLine($"Document:  {change.FullDocument?.Name}");
    Console.WriteLine($"Resume token: {change.ResumeToken}");
});
```

## Filtering Change Events

Use an aggregation pipeline to filter specific operation types:

```csharp
var pipeline = new EmptyPipelineDefinition<ChangeStreamDocument<Product>>()
    .Match(change =>
        change.OperationType == ChangeStreamOperationType.Insert ||
        change.OperationType == ChangeStreamOperationType.Update);

using var cursor = await products.WatchAsync(pipeline);

await cursor.ForEachAsync(change =>
    Console.WriteLine($"{change.OperationType}: {change.FullDocument?.Name}"));
```

## Receiving Full Documents on Update

By default, update events only contain the diff (`UpdateDescription`). Use `FullDocument = ChangeStreamFullDocumentOption.UpdateLookup` to also receive the current state:

```csharp
var options = new ChangeStreamOptions
{
    FullDocument = ChangeStreamFullDocumentOption.UpdateLookup
};

using var cursor = await products.WatchAsync(options: options);

await cursor.ForEachAsync(change =>
{
    if (change.OperationType == ChangeStreamOperationType.Update)
    {
        Console.WriteLine($"Updated fields: {change.UpdateDescription?.UpdatedFields}");
        Console.WriteLine($"Full document: {change.FullDocument?.Name}");
    }
});
```

## Resuming After Restart

Persist the resume token after processing each event:

```csharp
BsonDocument resumeToken = LoadResumeTokenFromStorage(); // your persistence logic

var options = new ChangeStreamOptions
{
    ResumeAfter = resumeToken,
    FullDocument = ChangeStreamFullDocumentOption.UpdateLookup
};

using var cursor = await products.WatchAsync(options: options);

await cursor.ForEachAsync(change =>
{
    ProcessChange(change);
    SaveResumeToken(change.ResumeToken); // persist after each event
});
```

## Running in a Background Service

Use a hosted background service to watch changes continuously:

```csharp
using Microsoft.Extensions.Hosting;

public class ChangeStreamWorker : BackgroundService
{
    private readonly IMongoCollection<Product> _products;

    public ChangeStreamWorker(IMongoDatabase database)
    {
        _products = database.GetCollection<Product>("products");
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var options = new ChangeStreamOptions
        {
            FullDocument = ChangeStreamFullDocumentOption.UpdateLookup
        };

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var cursor = await _products.WatchAsync(
                    options: options,
                    cancellationToken: stoppingToken);

                await cursor.ForEachAsync(change =>
                {
                    Console.WriteLine($"Change: {change.OperationType}");
                }, stoppingToken);
            }
            catch (OperationCanceledException) { break; }
            catch (MongoException ex)
            {
                Console.WriteLine($"Reconnecting after error: {ex.Message}");
                await Task.Delay(1000, stoppingToken);
            }
        }
    }
}
```

## Watching at Database and Cluster Level

```csharp
// Watch all collections in a database
using var dbCursor = await db.WatchAsync();

// Watch the entire cluster
using var clusterCursor = await client.WatchAsync();
```

## Summary

MongoDB Change Streams in the .NET Driver are accessed via the `WatchAsync()` method on collections, databases, or the client. Use pipeline definitions to filter operation types, `ChangeStreamFullDocumentOption.UpdateLookup` to receive full documents on updates, and resume tokens for fault-tolerant processing. Run the watcher in a `BackgroundService` for long-running change stream consumers in ASP.NET Core applications.
