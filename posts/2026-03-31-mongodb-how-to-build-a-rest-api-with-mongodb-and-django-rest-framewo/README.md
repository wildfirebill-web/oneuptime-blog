# How to Build a REST API with MongoDB and Django REST Framework

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Django, Django REST Framework, Python, REST API

Description: Learn how to build a REST API using Django REST Framework and djongo or pymongo to connect Django to MongoDB with serializers, views, and pagination.

---

## Project Setup

```bash
pip install django djangorestframework djongo pymongo
django-admin startproject myproject
cd myproject
python manage.py startapp users
```

## Django Settings for MongoDB

```python
# settings.py
INSTALLED_APPS = [
    'django.contrib.contenttypes',
    'django.contrib.auth',
    'rest_framework',
    'users',
]

DATABASES = {
    'default': {
        'ENGINE': 'djongo',
        'NAME': 'myapp',
        'ENFORCE_SCHEMA': False,
        'CLIENT': {
            'host': 'mongodb://localhost:27017',
        },
    }
}

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}
```

## Model Definition

```python
# users/models.py
from djongo import models


class User(models.Model):
    name = models.CharField(max_length=200)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = 'users'

    def __str__(self):
        return f'{self.name} <{self.email}>'
```

## Serializer

```python
# users/serializers.py
from rest_framework import serializers
from .models import User


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'name', 'email', 'created_at', 'updated_at']
        read_only_fields = ['id', 'created_at', 'updated_at']

    def validate_email(self, value):
        return value.strip().lower()
```

## Views

```python
# users/views.py
from rest_framework import generics, status
from rest_framework.response import Response
from django.db import IntegrityError
from .models import User
from .serializers import UserSerializer


class UserListCreateView(generics.ListCreateAPIView):
    queryset = User.objects.all().order_by('-created_at')
    serializer_class = UserSerializer

    def create(self, request, *args, **kwargs):
        try:
            return super().create(request, *args, **kwargs)
        except IntegrityError:
            return Response(
                {'error': 'Email already exists'},
                status=status.HTTP_409_CONFLICT
            )


class UserDetailView(generics.RetrieveUpdateDestroyAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    http_method_names = ['get', 'patch', 'delete']

    def destroy(self, request, *args, **kwargs):
        instance = self.get_object()
        instance.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

## URLs

```python
# users/urls.py
from django.urls import path
from .views import UserListCreateView, UserDetailView

urlpatterns = [
    path('', UserListCreateView.as_view(), name='user-list'),
    path('<int:pk>/', UserDetailView.as_view(), name='user-detail'),
]
```

```python
# myproject/urls.py
from django.urls import path, include

urlpatterns = [
    path('api/users/', include('users.urls')),
]
```

## Alternative: Using PyMongo Directly with DRF

For full MongoDB flexibility without the ORM, use PyMongo directly:

```python
# users/mongo_views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from pymongo import MongoClient
from bson import ObjectId
from bson.errors import InvalidId


client = MongoClient('mongodb://localhost:27017')
db = client['myapp']


class UserListView(APIView):
    def get(self, request):
        page = int(request.query_params.get('page', 1))
        limit = int(request.query_params.get('limit', 20))
        skip = (page - 1) * limit

        users = list(db.users.find({}).skip(skip).limit(limit))
        for u in users:
            u['_id'] = str(u['_id'])

        return Response({'data': users, 'page': page})

    def post(self, request):
        name = request.data.get('name')
        email = request.data.get('email')
        if not name or not email:
            return Response({'error': 'name and email required'}, status=400)

        result = db.users.insert_one({'name': name, 'email': email})
        return Response({'_id': str(result.inserted_id)}, status=201)
```

## Testing the API

```bash
# Create user
curl -X POST http://localhost:8000/api/users/ \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com"}'

# List users
curl "http://localhost:8000/api/users/?page=1"

# Partial update
curl -X PATCH http://localhost:8000/api/users/1/ \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice Smith"}'
```

## Summary

Django REST Framework integrates with MongoDB through djongo for ORM-style access or directly via PyMongo for full flexibility. The djongo approach lets you use standard DRF serializers, views, and pagination, while the PyMongo approach gives you direct access to aggregation pipelines and MongoDB-specific features without an ORM abstraction layer.
