# How to Use MySQL with Entity Framework Core

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Entity Framework, C#, ORM, .NET

Description: Learn how to configure Entity Framework Core with MySQL in a .NET application using Pomelo provider, define entities, run migrations, and query data.

---

## Introduction

Entity Framework Core (EF Core) is the primary ORM for .NET applications. MySQL support is provided by the community-maintained Pomelo.EntityFrameworkCore.MySql package, which is the recommended choice for EF Core with MySQL.

## Installing NuGet Packages

```bash
dotnet add package Pomelo.EntityFrameworkCore.MySql
dotnet add package Microsoft.EntityFrameworkCore.Design
```

## Configuring the Connection in Program.cs

```csharp
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseMySql(
        connectionString,
        ServerVersion.AutoDetect(connectionString),
        opts => opts.CommandTimeout(60)
    ));
```

In `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Port=3306;Database=mydb;User=root;Password=password;CharSet=utf8mb4;"
  }
}
```

## Defining the DbContext and Entities

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) {}

    public DbSet<Product> Products { get; set; }
    public DbSet<Category> Categories { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasIndex(p => new { p.CategoryId, p.Price });
            entity.Property(p => p.Price).HasPrecision(10, 2);
        });
    }
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public ICollection<Product> Products { get; set; } = new List<Product>();
}

public class Product
{
    public int Id { get; set; }
    public int CategoryId { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public Category Category { get; set; } = null!;
}
```

## Creating and Applying Migrations

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

## CRUD Operations

```csharp
// Inject via constructor
public class ProductService
{
    private readonly AppDbContext _db;
    public ProductService(AppDbContext db) => _db = db;

    // Create
    public async Task AddProductAsync(Product product)
    {
        _db.Products.Add(product);
        await _db.SaveChangesAsync();
    }

    // Read with eager loading
    public async Task<List<Product>> GetAffordableProductsAsync(decimal maxPrice)
    {
        return await _db.Products
            .Include(p => p.Category)
            .Where(p => p.Price <= maxPrice && p.Stock > 0)
            .OrderBy(p => p.Price)
            .ToListAsync();
    }

    // Update
    public async Task RestockLowInventoryAsync()
    {
        await _db.Products
            .Where(p => p.Stock < 5)
            .ExecuteUpdateAsync(s => s.SetProperty(p => p.Stock, 10));
    }

    // Delete
    public async Task DeleteExpensiveProductsAsync(decimal threshold)
    {
        await _db.Products
            .Where(p => p.Price > threshold)
            .ExecuteDeleteAsync();
    }
}
```

## Raw SQL Queries

```csharp
var results = await _db.Products
    .FromSqlRaw("SELECT * FROM products WHERE MATCH(name) AGAINST({0} IN BOOLEAN MODE)", searchTerm)
    .Include(p => p.Category)
    .ToListAsync();
```

## Summary

EF Core with MySQL via the Pomelo provider requires minimal setup - configure the connection string, define a `DbContext`, and use `dotnet ef migrations` to manage schema. Use LINQ queries with `Include` for eager loading, `ExecuteUpdateAsync` and `ExecuteDeleteAsync` for bulk operations, and `FromSqlRaw` for queries requiring MySQL-specific features like `MATCH AGAINST` full-text search.
