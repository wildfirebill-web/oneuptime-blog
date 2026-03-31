# How to Use MongoDB with ASP.NET Web API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Asp.Net, C#, Web Api, Database

Description: Step-by-step guide to integrating MongoDB into an ASP.NET Core Web API using the official .NET driver with dependency injection and CRUD operations.

---

## Why MongoDB with ASP.NET Core

MongoDB's flexible document model is a natural fit for REST APIs that evolve quickly. The official MongoDB .NET driver provides a typed, async-first client that integrates cleanly with ASP.NET Core's dependency injection and middleware pipeline.

## Installation

Create a Web API project and add the MongoDB driver:

```bash
dotnet new webapi -n ProductApi
cd ProductApi
dotnet add package MongoDB.Driver
```

## Configuration

Add your MongoDB settings to `appsettings.json`:

```json
{
  "MongoDB": {
    "ConnectionString": "mongodb://localhost:27017",
    "DatabaseName": "ProductDb"
  }
}
```

Create a settings class:

```csharp
// Settings/MongoDbSettings.cs
public class MongoDbSettings
{
    public string ConnectionString { get; set; } = string.Empty;
    public string DatabaseName { get; set; } = string.Empty;
}
```

## Registering the Client

Register MongoDB as a singleton in `Program.cs`:

```csharp
builder.Services.Configure<MongoDbSettings>(
    builder.Configuration.GetSection("MongoDB"));

builder.Services.AddSingleton<IMongoClient>(sp =>
{
    var settings = sp.GetRequiredService<IOptions<MongoDbSettings>>().Value;
    return new MongoClient(settings.ConnectionString);
});

builder.Services.AddSingleton<ProductService>();
```

## Defining the Model

```csharp
// Models/Product.cs
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;

public class Product
{
    [BsonId]
    [BsonRepresentation(BsonType.ObjectId)]
    public string? Id { get; set; }

    [BsonElement("name")]
    public string Name { get; set; } = string.Empty;

    [BsonElement("price")]
    public decimal Price { get; set; }

    [BsonElement("inStock")]
    public bool InStock { get; set; } = true;
}
```

## Service Layer

```csharp
// Services/ProductService.cs
public class ProductService
{
    private readonly IMongoCollection<Product> _products;

    public ProductService(IMongoClient client, IOptions<MongoDbSettings> settings)
    {
        var db = client.GetDatabase(settings.Value.DatabaseName);
        _products = db.GetCollection<Product>("products");
    }

    public async Task<List<Product>> GetAllAsync() =>
        await _products.Find(_ => true).SortBy(p => p.Name).ToListAsync();

    public async Task<Product?> GetByIdAsync(string id) =>
        await _products.Find(p => p.Id == id).FirstOrDefaultAsync();

    public async Task CreateAsync(Product product) =>
        await _products.InsertOneAsync(product);

    public async Task UpdateAsync(string id, Product updated) =>
        await _products.ReplaceOneAsync(p => p.Id == id, updated);

    public async Task DeleteAsync(string id) =>
        await _products.DeleteOneAsync(p => p.Id == id);
}
```

## Controller

```csharp
// Controllers/ProductsController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly ProductService _service;

    public ProductsController(ProductService service) => _service = service;

    [HttpGet]
    public async Task<IActionResult> Get() =>
        Ok(await _service.GetAllAsync());

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(string id)
    {
        var product = await _service.GetByIdAsync(id);
        return product is null ? NotFound() : Ok(product);
    }

    [HttpPost]
    public async Task<IActionResult> Create(Product product)
    {
        await _service.CreateAsync(product);
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(string id, Product product)
    {
        await _service.UpdateAsync(id, product);
        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(string id)
    {
        await _service.DeleteAsync(id);
        return NoContent();
    }
}
```

## Running the API

```bash
dotnet run
```

Test with curl:

```bash
curl -X POST http://localhost:5000/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"Widget","price":9.99,"inStock":true}'
```

## Summary

The MongoDB .NET driver pairs naturally with ASP.NET Core's dependency injection and async patterns. By defining a settings class, registering `IMongoClient` as a singleton, and encapsulating queries in a service layer, you get a clean, testable architecture for MongoDB-backed Web APIs.
