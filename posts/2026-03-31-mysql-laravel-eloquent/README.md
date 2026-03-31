# How to Use MySQL with Laravel Eloquent

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Laravel, Eloquent, PHP, ORM

Description: Learn how to connect Laravel to MySQL and use Eloquent ORM to define models, write migrations, build relationships, and query data efficiently.

---

## Introduction

Laravel's Eloquent ORM provides an expressive, fluent interface for MySQL. Each database table has a corresponding model class, and Eloquent handles relationships, scopes, mutators, and query building while generating optimized SQL under the hood.

## Configuring MySQL in .env

```text
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=mydb
DB_USERNAME=laravel_user
DB_PASSWORD=password
```

`config/database.php` reads these values automatically:

```php
'mysql' => [
    'driver'    => 'mysql',
    'host'      => env('DB_HOST', '127.0.0.1'),
    'port'      => env('DB_PORT', '3306'),
    'database'  => env('DB_DATABASE'),
    'username'  => env('DB_USERNAME'),
    'password'  => env('DB_PASSWORD'),
    'charset'   => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
],
```

## Creating a Migration

```bash
php artisan make:migration create_products_table
```

```php
public function up(): void
{
    Schema::create('products', function (Blueprint $table) {
        $table->id();
        $table->foreignId('category_id')->constrained()->cascadeOnDelete();
        $table->string('name');
        $table->decimal('price', 10, 2);
        $table->integer('stock')->default(0);
        $table->boolean('active')->default(true);
        $table->timestamps();

        $table->index(['category_id', 'price']);
    });
}
```

```bash
php artisan migrate
```

## Defining an Eloquent Model

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Product extends Model
{
    protected $fillable = ['category_id', 'name', 'price', 'stock', 'active'];

    protected $casts = [
        'price' => 'decimal:2',
        'active' => 'boolean',
    ];

    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);
    }

    public function scopeActive($query)
    {
        return $query->where('active', true);
    }

    public function scopeInStock($query)
    {
        return $query->where('stock', '>', 0);
    }
}
```

## Querying with Eloquent

```php
use App\Models\Product;

// Create
$product = Product::create([
    'category_id' => 1,
    'name' => 'Laptop',
    'price' => 999.99,
    'stock' => 50,
]);

// Read with eager loading
$products = Product::with('category')
    ->active()
    ->inStock()
    ->where('price', '<', 1000)
    ->orderBy('price')
    ->paginate(20);

// Update
Product::where('stock', 0)->update(['active' => false]);

// Delete
Product::where('price', '>', 50000)->delete();
```

## Using the Query Builder

For complex queries beyond Eloquent:

```php
use Illuminate\Support\Facades\DB;

$results = DB::table('products')
    ->join('categories', 'products.category_id', '=', 'categories.id')
    ->select('categories.name as category', DB::raw('SUM(products.stock) as total_stock'))
    ->groupBy('categories.name')
    ->having('total_stock', '>', 0)
    ->get();
```

## Summary

Laravel Eloquent with MySQL is configured through the `.env` file and `config/database.php`. Define models with `$fillable`, `$casts`, and relationship methods, then query using Eloquent's fluent chain API. Use `with()` for eager loading to prevent N+1 queries, local scopes for reusable query logic, and the Query Builder for complex aggregations or joins that Eloquent cannot express cleanly.
