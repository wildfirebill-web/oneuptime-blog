# How to Set Up a PHP/Laravel Development Environment with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, PHP, Laravel, Development Environment, Docker, Xdebug, Composer

Description: Learn how to set up a PHP Laravel development environment with Xdebug and hot-reload in a Docker container managed by Portainer.

---

Running Laravel development in Docker via Portainer ensures your team uses the same PHP version, extensions, and tools. Xdebug integration enables breakpoint debugging from VS Code or PhpStorm.

## Dev Environment Compose Stack

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: laravel_dev
      MYSQL_USER: laravel
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"

  redis:
    image: redis:7-alpine
    restart: unless-stopped

  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    restart: unless-stopped
    depends_on:
      - mysql
      - redis
    ports:
      - "8000:8000"    # App
      - "9003:9003"    # Xdebug
    environment:
      APP_ENV: local
      DB_HOST: mysql
      DB_DATABASE: laravel_dev
      DB_USERNAME: laravel
      DB_PASSWORD: secret
      REDIS_HOST: redis
      XDEBUG_MODE: develop,debug
      XDEBUG_CONFIG: client_host=host.docker.internal
    volumes:
      - ./app:/app
    working_dir: /app
    command: php artisan serve --host=0.0.0.0 --port=8000

volumes:
  mysql_data:
```

## Development Dockerfile

```dockerfile
# Dockerfile.dev

FROM php:8.3-cli-alpine

RUN apk add --no-cache \
    git curl zip unzip libzip-dev oniguruma-dev \
    && docker-php-ext-install pdo pdo_mysql zip mbstring

# Install Xdebug
RUN pecl install xdebug && docker-php-ext-enable xdebug

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /app
```

## Xdebug Configuration

Create `php/xdebug.ini`:

```ini
[xdebug]
zend_extension=xdebug.so
xdebug.mode=develop,debug
xdebug.start_with_request=yes
xdebug.client_host=host.docker.internal
xdebug.client_port=9003
xdebug.idekey=VSCODE
```

## VS Code PHP Debug Configuration

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Listen for Xdebug",
      "type": "php",
      "request": "launch",
      "port": 9003,
      "pathMappings": {
        "/app": "${workspaceFolder}/app"
      }
    }
  ]
}
```
