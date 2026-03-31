# How to Use MongoDB with Symfony and Doctrine MongoDB ODM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Symfony, Doctrine, ODM, PHP

Description: Learn how to integrate MongoDB with Symfony using Doctrine MongoDB ODM for annotation-driven document mapping and repository-based data access.

---

## Overview

Doctrine MongoDB ODM (Object Document Mapper) is the most popular way to use MongoDB with Symfony. It mirrors Doctrine ORM's API - using annotations or PHP attributes to map PHP classes to MongoDB collections, and providing a `DocumentManager` for persistence. This allows Symfony developers to work with MongoDB using patterns they already know from relational Doctrine ORM.

## Installation

```bash
composer require doctrine/mongodb-odm-bundle
```

## Bundle Registration

```php
// config/bundles.php
return [
    // ...
    Doctrine\Bundle\MongoDBBundle\DoctrineMongoDBBundle::class => ['all' => true],
];
```

## Configuration

```yaml
# config/packages/doctrine_mongodb.yaml
doctrine_mongodb:
  connections:
    default:
      server: '%env(MONGODB_URL)%'
  default_database: '%env(MONGODB_DB)%'
  document_managers:
    default:
      auto_mapping: true
```

```bash
# .env
MONGODB_URL=mongodb://localhost:27017
MONGODB_DB=shopdb
```

## Defining a Document

```php
// src/Document/Product.php
namespace App\Document;

use Doctrine\ODM\MongoDB\Mapping\Annotations as MongoDB;

#[MongoDB\Document(collection: 'products')]
#[MongoDB\Index(keys: ['category' => 'asc'])]
class Product
{
    #[MongoDB\Id]
    private ?string $id = null;

    #[MongoDB\Field(type: 'string')]
    private string $name;

    #[MongoDB\Field(type: 'float')]
    private float $price;

    #[MongoDB\Field(type: 'string')]
    private string $category;

    #[MongoDB\Field(type: 'int')]
    private int $stock;

    // constructor
    public function __construct(string $name, float $price, string $category, int $stock)
    {
        $this->name = $name;
        $this->price = $price;
        $this->category = $category;
        $this->stock = $stock;
    }

    // getters/setters
    public function getId(): ?string { return $this->id; }
    public function getName(): string { return $this->name; }
    public function getPrice(): float { return $this->price; }
    public function setPrice(float $price): void { $this->price = $price; }
    public function getCategory(): string { return $this->category; }
    public function getStock(): int { return $this->stock; }
    public function setStock(int $stock): void { $this->stock = $stock; }
}
```

## Creating a Repository

```php
// src/Repository/ProductRepository.php
namespace App\Repository;

use App\Document\Product;
use Doctrine\Bundle\MongoDBBundle\Repository\ServiceDocumentRepository;
use Doctrine\Persistence\ManagerRegistry;

class ProductRepository extends ServiceDocumentRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Product::class);
    }

    public function findByCategory(string $category): array
    {
        return $this->createQueryBuilder()
            ->field('category')->equals($category)
            ->sort('price', 'ASC')
            ->getQuery()
            ->execute()
            ->toArray();
    }

    public function findAffordable(float $maxPrice): array
    {
        return $this->createQueryBuilder()
            ->field('price')->lte($maxPrice)
            ->field('stock')->gt(0)
            ->getQuery()
            ->execute()
            ->toArray();
    }
}
```

## Using DocumentManager in a Service

```php
namespace App\Service;

use App\Document\Product;
use App\Repository\ProductRepository;
use Doctrine\ODM\MongoDB\DocumentManager;

class ProductService
{
    public function __construct(
        private DocumentManager $dm,
        private ProductRepository $repository,
    ) {}

    public function create(string $name, float $price, string $category, int $stock): Product
    {
        $product = new Product($name, $price, $category, $stock);
        $this->dm->persist($product);
        $this->dm->flush();
        return $product;
    }

    public function updatePrice(string $id, float $price): void
    {
        $product = $this->dm->find(Product::class, $id);
        if (!$product) {
            throw new \RuntimeException("Product not found");
        }
        $product->setPrice($price);
        $this->dm->flush(); // no need to call persist() again
    }

    public function delete(string $id): void
    {
        $product = $this->dm->find(Product::class, $id);
        if ($product) {
            $this->dm->remove($product);
            $this->dm->flush();
        }
    }
}
```

## Embedded Documents

```php
#[MongoDB\EmbeddedDocument]
class Address
{
    #[MongoDB\Field(type: 'string')]
    private string $street;

    #[MongoDB\Field(type: 'string')]
    private string $city;
}

#[MongoDB\Document]
class Customer
{
    #[MongoDB\Id]
    private ?string $id = null;

    #[MongoDB\EmbedOne(targetDocument: Address::class)]
    private Address $shippingAddress;
}
```

## Summary

Doctrine MongoDB ODM with Symfony provides a full ORM-like experience for MongoDB. Use PHP attributes (`#[MongoDB\Document]`, `#[MongoDB\Field]`, `#[MongoDB\EmbedOne]`) to map document classes, extend `ServiceDocumentRepository` for type-safe repository methods with the query builder, and inject `DocumentManager` for persist/remove/flush operations. The unit-of-work pattern means you call `flush()` once after all mutations in a request to batch database writes.
