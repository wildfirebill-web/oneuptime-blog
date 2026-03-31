# How to Use MongoDB with Django REST Framework

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Django, REST, PyMongo, Serializer

Description: Learn how to build a Django REST Framework API backed by MongoDB using PyMongo directly, bypassing the ORM with custom serializers and views.

---

## Introduction

Django REST Framework (DRF) is the most popular toolkit for building REST APIs in Django. Although Django's built-in ORM does not support MongoDB, you can combine DRF's serializers, views, and authentication with direct PyMongo queries. This guide shows how to wire the two together cleanly.

## Installation

```bash
pip install django djangorestframework pymongo
```

`settings.py`:

```python
INSTALLED_APPS = [
    # ...
    'rest_framework',
]

MONGODB_URI = 'mongodb://localhost:27017'
MONGODB_DB  = 'shop'
```

## MongoDB Client Singleton

```python
# myapp/db.py
from django.conf import settings
from pymongo import MongoClient

_client = None

def get_db():
    global _client
    if _client is None:
        _client = MongoClient(settings.MONGODB_URI)
    return _client[settings.MONGODB_DB]
```

## Serializers

```python
# myapp/serializers.py
from rest_framework import serializers

class ProductSerializer(serializers.Serializer):
    id       = serializers.CharField(read_only=True, source='_id')
    name     = serializers.CharField(max_length=200)
    price    = serializers.FloatField(min_value=0)
    category = serializers.CharField(max_length=100)
    inStock  = serializers.BooleanField(default=True)
```

## Views

```python
# myapp/views.py
from rest_framework.views     import APIView
from rest_framework.response  import Response
from rest_framework           import status
from bson.objectid            import ObjectId
from .db                      import get_db
from .serializers             import ProductSerializer

class ProductListView(APIView):
    def get(self, request):
        db       = get_db()
        category = request.query_params.get('category')
        query    = {'category': category} if category else {}
        products = list(db.products.find(query))
        for p in products:
            p['_id'] = str(p['_id'])
        return Response(products)

    def post(self, request):
        serializer = ProductSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        db     = get_db()
        result = db.products.insert_one(serializer.validated_data)
        return Response({'id': str(result.inserted_id)}, status=status.HTTP_201_CREATED)


class ProductDetailView(APIView):
    def get(self, request, pk):
        db      = get_db()
        product = db.products.find_one({'_id': ObjectId(pk)})
        if not product:
            return Response({'error': 'Not found'}, status=status.HTTP_404_NOT_FOUND)
        product['_id'] = str(product['_id'])
        return Response(product)

    def put(self, request, pk):
        serializer = ProductSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        db = get_db()
        db.products.update_one(
            {'_id': ObjectId(pk)},
            {'$set': serializer.validated_data}
        )
        return Response({'updated': True})

    def delete(self, request, pk):
        db = get_db()
        db.products.delete_one({'_id': ObjectId(pk)})
        return Response(status=status.HTTP_204_NO_CONTENT)
```

## URL Configuration

```python
# myapp/urls.py
from django.urls import path
from .views import ProductListView, ProductDetailView

urlpatterns = [
    path('products/',      ProductListView.as_view()),
    path('products/<pk>/', ProductDetailView.as_view()),
]
```

## Adding DRF Authentication

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

## Summary

Using DRF with MongoDB bypasses Django's ORM and uses PyMongo directly. A singleton `get_db()` function returns the database connection, `APIView` subclasses handle HTTP verbs, and DRF `Serializer` classes validate and parse request data. ObjectId fields are converted to strings before returning JSON responses, since the standard JSON encoder cannot serialize `bson.ObjectId`.
