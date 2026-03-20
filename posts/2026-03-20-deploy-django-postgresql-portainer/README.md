# How to Deploy a Django + PostgreSQL Stack via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Django, PostgreSQL, Python, Docker Compose, Web Framework

Description: Learn how to deploy a Django web application with PostgreSQL via Portainer, including database migrations, static files, and production configuration.

---

Django with PostgreSQL is the standard production stack for Python web applications. Portainer simplifies managing the multi-container deployment with health checks and log visibility.

## Compose Stack

```yaml
version: "3.8"

services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: djangoapp
      POSTGRES_USER: django
      POSTGRES_PASSWORD: djangopass       # Change this
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U django"]
      interval: 10s
      timeout: 5s
      retries: 5

  web:
    image: python:3.12-slim
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy        # Wait until Postgres is ready
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://django:djangopass@db:5432/djangoapp
      DJANGO_SECRET_KEY: changeme-50-char-random-string
      DJANGO_ALLOWED_HOSTS: "*"
      DJANGO_DEBUG: "False"
    volumes:
      - ./app:/app
      - static_files:/app/staticfiles
    working_dir: /app
    command: >
      sh -c "pip install -r requirements.txt &&
             python manage.py migrate &&
             python manage.py collectstatic --noinput &&
             gunicorn config.wsgi:application --bind 0.0.0.0:8000"

volumes:
  postgres_data:
  static_files:
```

## Django Settings for Containers

Configure Django to read settings from environment variables:

```python
# config/settings.py
import os
import dj_database_url

SECRET_KEY = os.environ['DJANGO_SECRET_KEY']
DEBUG = os.environ.get('DJANGO_DEBUG', 'False') == 'True'
ALLOWED_HOSTS = os.environ.get('DJANGO_ALLOWED_HOSTS', 'localhost').split(',')

# Parse DATABASE_URL environment variable
DATABASES = {
    'default': dj_database_url.parse(os.environ['DATABASE_URL'])
}

# Serve static files from STATIC_ROOT
STATIC_ROOT = '/app/staticfiles'
```

## Running Management Commands

```bash
# In Portainer, go to Containers > web > Exec
# Or via docker exec:
docker exec -it <web-container> python manage.py createsuperuser
docker exec -it <web-container> python manage.py shell
```

## Monitoring

Use OneUptime to monitor `http://<host>:8000/` for HTTP 200. For Django health checks, install `django-health-check` and monitor `/health/` to verify database connectivity.
