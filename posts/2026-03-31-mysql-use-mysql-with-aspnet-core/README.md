# How to Use MySQL with ASP.NET Core

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, ASP.NET, Entity Framework

Description: Integrate MySQL with ASP.NET Core using Pomelo Entity Framework Core provider, configure DbContext, and run EF Core migrations.

---

ASP.NET Core connects to MySQL through the Pomelo Entity Framework Core provider - the most widely used and actively maintained MySQL connector for EF Core. It supports EF Core 8, async queries, and standard code-first migrations.

## Installing the Package

```bash
dotnet add package Pomelo.EntityFrameworkCore.MySql
dotnet add package Microsoft.EntityFrameworkCore.Design
```

## Connection String in appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=127.0.0.1;Port=3306;Database=app_db;User=app_user;Password=strongpassword;"
  }
}
```

## Registering the DbContext

In `Program.cs`:

```csharp
using Pomelo.EntityFrameworkCore.MySql.Infrastructure;

var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
var serverVersion   = new MySqlServerVersion(new Version(8, 0, 36));

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseMySql(connectionString, serverVersion, mySqlOptions =>
        mySqlOptions.EnableRetryOnFailure(maxRetryCount: 3))
);
```

`EnableRetryOnFailure` handles transient connection errors automatically.

## Defining the DbContext and Entities

```csharp
// Models/Product.cs
public class Product
{
    public int     Id        { get; set; }
    public string  Name      { get; set; } = string.Empty;
    public decimal Price     { get; set; }
    public DateTime CreatedAt { get; set; }
}

// Data/AppDbContext.cs
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) {}

    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>(entity =>
        {
            entity.Property(e => e.Name).HasMaxLength(200).IsRequired();
            entity.Property(e => e.Price).HasPrecision(10, 2);
            entity.Property(e => e.CreatedAt).HasDefaultValueSql("CURRENT_TIMESTAMP");
        });
    }
}
```

## Running Migrations

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

## Querying the Database

```csharp
// Controllers/ProductsController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly AppDbContext _db;

    public ProductsController(AppDbContext db) => _db = db;

    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var products = await _db.Products
            .OrderByDescending(p => p.Id)
            .Take(50)
            .ToListAsync();
        return Ok(products);
    }

    [HttpPost]
    public async Task<IActionResult> Create(Product product)
    {
        _db.Products.Add(product);
        await _db.SaveChangesAsync();
        return CreatedAtAction(nameof(GetAll), product);
    }
}
```

## Summary

ASP.NET Core connects to MySQL cleanly via Pomelo EF Core. Use `EnableRetryOnFailure` for resilience, define precision on decimal columns to avoid schema mismatches, and always run migrations via `dotnet ef` rather than letting `EnsureCreated` manage schema in production.
