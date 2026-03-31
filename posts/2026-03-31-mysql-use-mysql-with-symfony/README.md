# How to Use MySQL with Symfony

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Symfony, Doctrine

Description: Connect Symfony to MySQL using Doctrine ORM, configure the DATABASE_URL env variable, define entities, and run Doctrine migrations.

---

Symfony uses Doctrine ORM as its default database abstraction layer. Doctrine provides entity mapping, a query builder, native SQL support, and a migration tool - all of which work well with MySQL 8.

## Prerequisites

Install Symfony with Doctrine support:

```bash
composer create-project symfony/skeleton my-project
cd my-project
composer require symfony/orm-pack
composer require --dev symfony/maker-bundle
```

## Configuring DATABASE_URL

Set the connection string in `.env` (or `.env.local` for local overrides):

```text
DATABASE_URL="mysql://app_user:strongpassword@127.0.0.1:3306/app_db?serverVersion=8.0&charset=utf8mb4"
```

Always specify `serverVersion` so Doctrine generates MySQL-version-compatible SQL.

## config/packages/doctrine.yaml

```yaml
doctrine:
    dbal:
        url: '%env(DATABASE_URL)%'
        options:
            1002: "SET NAMES utf8mb4"
    orm:
        auto_generate_proxy_classes: true
        naming_strategy: doctrine.orm.naming_strategy.underscore_number_aware
        auto_mapping: true
        mappings:
            App:
                type: attribute
                dir: '%kernel.project_dir%/src/Entity'
                prefix: 'App\Entity'
```

## Generating an Entity

```bash
php bin/console make:entity Product
```

This creates `src/Entity/Product.php` with Doctrine attributes:

```php
#[ORM\Entity(repositoryClass: ProductRepository::class)]
class Product
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 200)]
    private string $name;

    #[ORM\Column(type: 'decimal', precision: 10, scale: 2)]
    private string $price;
}
```

## Running Migrations

```bash
php bin/console doctrine:migrations:diff
php bin/console doctrine:migrations:migrate
```

## Querying with the Repository

```php
// ProductRepository.php
public function findCheapProducts(float $maxPrice): array
{
    return $this->createQueryBuilder('p')
        ->where('p.price <= :price')
        ->setParameter('price', $maxPrice)
        ->orderBy('p.price', 'ASC')
        ->setMaxResults(20)
        ->getQuery()
        ->getResult();
}
```

## Using Native SQL

```php
$conn = $this->getEntityManager()->getConnection();
$sql  = 'SELECT id, name FROM products WHERE price < :price LIMIT 10';
$rows = $conn->fetchAllAssociative($sql, ['price' => 50.0]);
```

## Summary

Symfony and MySQL connect through Doctrine, which handles entity mapping, migration generation, and query building. Always specify `serverVersion` in `DATABASE_URL`, use `utf8mb4` charset, and run all schema changes through Doctrine migrations rather than manual SQL in production.
