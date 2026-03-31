# How to Use MongoDB with ASP.NET Core

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, ASP.NET Core, CSharp, DotNet, REST API

Description: Learn how to integrate MongoDB into an ASP.NET Core application using the .NET Driver, repository pattern, and dependency injection.

---

## Overview

ASP.NET Core and MongoDB are a natural pairing for building modern REST APIs. The MongoDB .NET Driver integrates cleanly with ASP.NET Core's dependency injection system. This guide covers project setup, DI registration, a repository pattern, and a complete CRUD controller.

## Setup

```bash
dotnet new webapi -n ShopApi
cd ShopApi
dotnet add package MongoDB.Driver
```

## Configuration

```json
// appsettings.json
{
  "MongoDB": {
    "ConnectionString": "mongodb://localhost:27017",
    "DatabaseName": "shopdb"
  }
}
```

```csharp
// MongoDbSettings.cs
public class MongoDbSettings
{
    public string ConnectionString { get; set; } = null!;
    public string DatabaseName { get; set; } = null!;
}
```

## Registering MongoDB with DI

```csharp
// Program.cs
using MongoDB.Driver;

builder.Services.Configure<MongoDbSettings>(
    builder.Configuration.GetSection("MongoDB"));

builder.Services.AddSingleton<IMongoClient>(sp =>
{
    var settings = sp.GetRequiredService<IOptions<MongoDbSettings>>().Value;
    return new MongoClient(settings.ConnectionString);
});

builder.Services.AddScoped<IMongoDatabase>(sp =>
{
    var settings = sp.GetRequiredService<IOptions<MongoDbSettings>>().Value;
    return sp.GetRequiredService<IMongoClient>()
             .GetDatabase(settings.DatabaseName);
});

builder.Services.AddScoped<IProductRepository, ProductRepository>();
```

## Document Model

```csharp
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;

public class Product
{
    [BsonId]
    [BsonRepresentation(BsonType.ObjectId)]
    public string? Id { get; set; }
    public string Name { get; set; } = null!;
    public double Price { get; set; }
    public string Category { get; set; } = null!;
    public int Stock { get; set; }
}
```

## Repository Interface and Implementation

```csharp
public interface IProductRepository
{
    Task<List<Product>> GetAllAsync();
    Task<Product?> GetByIdAsync(string id);
    Task<Product> CreateAsync(Product product);
    Task UpdateAsync(string id, Product product);
    Task DeleteAsync(string id);
}

public class ProductRepository : IProductRepository
{
    private readonly IMongoCollection<Product> _collection;

    public ProductRepository(IMongoDatabase database)
    {
        _collection = database.GetCollection<Product>("products");
    }

    public async Task<List<Product>> GetAllAsync() =>
        await _collection.Find(_ => true).ToListAsync();

    public async Task<Product?> GetByIdAsync(string id) =>
        await _collection.Find(p => p.Id == id).FirstOrDefaultAsync();

    public async Task<Product> CreateAsync(Product product)
    {
        await _collection.InsertOneAsync(product);
        return product;
    }

    public async Task UpdateAsync(string id, Product product) =>
        await _collection.ReplaceOneAsync(p => p.Id == id, product);

    public async Task DeleteAsync(string id) =>
        await _collection.DeleteOneAsync(p => p.Id == id);
}
```

## REST Controller

```csharp
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repo;

    public ProductsController(IProductRepository repo) => _repo = repo;

    [HttpGet]
    public async Task<ActionResult<List<Product>>> GetAll() =>
        Ok(await _repo.GetAllAsync());

    [HttpGet("{id}")]
    public async Task<ActionResult<Product>> GetById(string id)
    {
        var product = await _repo.GetByIdAsync(id);
        return product is null ? NotFound() : Ok(product);
    }

    [HttpPost]
    public async Task<ActionResult<Product>> Create(Product product)
    {
        var created = await _repo.CreateAsync(product);
        return CreatedAtAction(nameof(GetById),
            new { id = created.Id }, created);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(string id, Product product)
    {
        var existing = await _repo.GetByIdAsync(id);
        if (existing is null) return NotFound();

        product.Id = id;
        await _repo.UpdateAsync(id, product);
        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(string id)
    {
        var existing = await _repo.GetByIdAsync(id);
        if (existing is null) return NotFound();

        await _repo.DeleteAsync(id);
        return NoContent();
    }
}
```

## Summary

Integrating MongoDB with ASP.NET Core requires registering `MongoClient` as a singleton and `IMongoDatabase` as scoped in the DI container. Place connection settings in `appsettings.json`, bind them to a typed settings class, and inject `IMongoDatabase` into repository classes. The repository pattern keeps MongoDB details out of controllers and makes unit testing straightforward by mocking the repository interface.
