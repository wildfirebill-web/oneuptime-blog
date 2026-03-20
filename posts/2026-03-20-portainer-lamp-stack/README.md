# How to Deploy a LAMP Stack via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, LAMP, Apache, MySQL, PHP, Docker Compose, Web Development

Description: Deploy a complete LAMP (Linux, Apache, MySQL, PHP) stack using Docker Compose through Portainer, with persistent storage, PHP configuration, and phpMyAdmin for database management.

## Introduction

The LAMP stack (Linux, Apache, MySQL, PHP) is one of the most widely deployed web application environments. Using Portainer and Docker Compose, you can spin up a complete, isolated LAMP environment in minutes with persistent data volumes and a web-based database manager. This guide walks through deploying LAMP via Portainer Stacks.

## Prerequisites

- Portainer CE or BE installed and running
- Docker Engine 20.10+
- At least 1 GB of available RAM

## Step 1: Open Stacks in Portainer

Navigate to your Portainer environment and go to **Stacks** → **Add Stack** → **Web Editor**. Name the stack `lamp`.

## Step 2: Write the Docker Compose File

Paste the following compose file into the Web Editor:

```yaml
version: "3.8"

services:
  # Apache + PHP web server
  web:
    image: php:8.2-apache
    container_name: lamp-web
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./www:/var/www/html          # Application files
      - ./apache/vhost.conf:/etc/apache2/sites-available/000-default.conf
    environment:
      APACHE_DOCUMENT_ROOT: /var/www/html
    depends_on:
      - db
    networks:
      - lamp-net

  # MySQL database
  db:
    image: mysql:8.0
    container_name: lamp-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: lampapp
      MYSQL_USER: lampuser
      MYSQL_PASSWORD: lamppassword
    volumes:
      - db_data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d  # SQL init scripts
    networks:
      - lamp-net

  # phpMyAdmin for database management
  phpmyadmin:
    image: phpmyadmin/phpmyadmin:latest
    container_name: lamp-phpmyadmin
    restart: unless-stopped
    ports:
      - "8081:80"
    environment:
      PMA_HOST: db
      PMA_PORT: 3306
      PMA_USER: root
      PMA_PASSWORD: rootpassword
    depends_on:
      - db
    networks:
      - lamp-net

volumes:
  db_data:
    driver: local

networks:
  lamp-net:
    driver: bridge
```

## Step 3: Configure PHP Extensions

The base `php:8.2-apache` image lacks common extensions. Create a custom Dockerfile or use a pre-built image with extensions:

```yaml
# Alternative: use a Dockerfile for the web service

  web:
    build:
      context: ./docker/php
      dockerfile: Dockerfile
    # rest of config...
```

```dockerfile
# docker/php/Dockerfile
FROM php:8.2-apache

# Install common PHP extensions
RUN docker-php-ext-install pdo pdo_mysql mysqli

# Install additional extensions via pecl
RUN pecl install redis && docker-php-ext-enable redis

# Enable Apache modules
RUN a2enmod rewrite headers

# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- \
    --install-dir=/usr/local/bin --filename=composer
```

## Step 4: Create Apache Virtual Host Config

```apache
# apache/vhost.conf
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Enable .htaccess rewrites (required for frameworks like Laravel)
    <IfModule mod_rewrite.c>
        RewriteEngine On
    </IfModule>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

## Step 5: Create a Test PHP Application

```php
<?php
// www/index.php - test connectivity to MySQL

$host = getenv('MYSQL_HOST') ?: 'db';
$dbname = getenv('MYSQL_DATABASE') ?: 'lampapp';
$user = getenv('MYSQL_USER') ?: 'lampuser';
$password = getenv('MYSQL_PASSWORD') ?: 'lamppassword';

try {
    $pdo = new PDO("mysql:host=$host;dbname=$dbname", $user, $password);
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    echo "<h1>LAMP Stack Running</h1>";
    echo "<p>Connected to MySQL successfully!</p>";
    echo "<p>PHP Version: " . phpversion() . "</p>";
} catch (PDOException $e) {
    echo "<h1>Connection failed: " . $e->getMessage() . "</h1>";
}
?>
```

## Step 6: Deploy and Verify

```bash
# After clicking Deploy in Portainer:

# Verify containers are running
docker ps | grep lamp

# Test the web application
curl http://localhost:8080

# Access phpMyAdmin at http://localhost:8081
# Login: root / rootpassword

# Check PHP info
curl http://localhost:8080/phpinfo.php  # if you created one

# View MySQL logs from Portainer:
# Containers → lamp-db → Logs

# Connect to MySQL CLI
docker exec -it lamp-db mysql -u lampuser -plamppassword lampapp
```

## Step 7: Set Environment Variables via Portainer UI

Instead of hardcoding credentials, use Portainer's environment variable feature:

1. In the Stack editor, add **Environment Variables**:
   - `MYSQL_ROOT_PASSWORD` → `your-root-password`
   - `MYSQL_PASSWORD` → `your-app-password`

2. Reference them in the compose file:

```yaml
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: lampapp
      MYSQL_USER: lampuser
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

## Step 8: Enable HTTPS with a Self-Signed Certificate

```yaml
  # Add to the web service
  web:
    ports:
      - "8080:80"
      - "8443:443"
    volumes:
      - ./ssl:/etc/apache2/ssl  # Place cert.pem and key.pem here
      - ./apache/ssl-vhost.conf:/etc/apache2/sites-available/default-ssl.conf
```

```apache
# apache/ssl-vhost.conf
<VirtualHost *:443>
    DocumentRoot /var/www/html
    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/cert.pem
    SSLCertificateKeyFile /etc/apache2/ssl/key.pem

    <Directory /var/www/html>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

## Conclusion

Deploying a LAMP stack via Portainer gives you a self-contained, reproducible development or production environment with MySQL data persisted in named volumes. phpMyAdmin is included for easy database administration through the browser. For production deployments, move credentials to Portainer's environment variable store or Docker secrets, configure proper SSL certificates, and restrict phpMyAdmin's network access. The containerized LAMP stack is easily updated by redeploying the stack with updated image tags.
