# How to Use MySQL with Django ORM

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Django, ORM, Python, Web Framework

Description: Learn how to configure Django to use MySQL as its database backend, define models, run migrations, and perform efficient queries with the Django ORM.

---

## Introduction

Django's ORM supports MySQL through the `mysqlclient` or `mysql-connector-python` drivers. Once configured, you define Python model classes and Django handles SQL generation, schema migrations, and connection management automatically.

## Installing Dependencies

```bash
pip install django mysqlclient
# or
pip install django mysql-connector-python
```

On Ubuntu, ensure the MySQL development headers are present:

```bash
sudo apt install libmysqlclient-dev
```

## Configuring MySQL in settings.py

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'mydb',
        'USER': 'django_user',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '3306',
        'OPTIONS': {
            'charset': 'utf8mb4',
            'init_command': "SET sql_mode='STRICT_TRANS_TABLES'",
        },
    }
}
```

## Creating a MySQL User for Django

```sql
CREATE USER 'django_user'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON mydb.* TO 'django_user'@'localhost';
FLUSH PRIVILEGES;
```

## Defining Models

```python
from django.db import models

class Category(models.Model):
    name = models.CharField(max_length=100)
    slug = models.SlugField(unique=True)

    class Meta:
        db_table = 'categories'

class Product(models.Model):
    category = models.ForeignKey(Category, on_delete=models.CASCADE)
    name = models.CharField(max_length=200)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = 'products'
        indexes = [
            models.Index(fields=['category', 'price']),
        ]
```

## Running Migrations

```bash
python manage.py makemigrations
python manage.py migrate
```

## Performing CRUD Operations

```python
from myapp.models import Category, Product

# Create
electronics = Category.objects.create(name='Electronics', slug='electronics')
Product.objects.create(
    category=electronics,
    name='Laptop',
    price=999.99,
    stock=50
)

# Read
products = Product.objects.filter(
    category__slug='electronics',
    price__lt=1000
).select_related('category').order_by('price')

# Update
Product.objects.filter(stock=0).update(stock=10)

# Delete
Product.objects.filter(price__gt=5000).delete()
```

## Using Raw SQL When Needed

```python
from django.db import connection

def get_low_stock_products(threshold=5):
    with connection.cursor() as cursor:
        cursor.execute(
            "SELECT id, name, stock FROM products WHERE stock <= %s ORDER BY stock",
            [threshold]
        )
        return cursor.fetchall()
```

## Optimizing with select_related and prefetch_related

```python
# Avoids N+1 queries - single JOIN query
products = Product.objects.select_related('category').all()

# For many-to-many or reverse FK
from myapp.models import Order
orders = Order.objects.prefetch_related('items').filter(status='pending')
```

## Summary

Django integrates with MySQL through the `mysqlclient` driver and a simple `DATABASES` configuration. Define models as Python classes, generate migrations with `makemigrations`, and use the ORM's query API for filtering, aggregation, and joins. Use `select_related` and `prefetch_related` to prevent N+1 query patterns, and drop to raw SQL for complex queries that the ORM cannot express efficiently.
