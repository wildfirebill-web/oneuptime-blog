# How to Deploy Apache HTTP Server via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Apache, HTTP Server, Docker, Web Server

Description: Learn how to deploy Apache HTTP Server (httpd) via Portainer with custom virtual hosts, SSL configuration, and persistent document root volumes.

## Apache via Portainer Stack

**Stacks → Add Stack → apache**

```yaml
version: "3.8"

services:
  apache:
    image: httpd:2.4-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      # Website content
      - apache_html:/usr/local/apache2/htdocs
      # Custom configuration
      - ./httpd-conf:/usr/local/apache2/conf/extra:ro
      # SSL certificates
      - ./ssl:/usr/local/apache2/ssl:ro
      # Access and error logs
      - apache_logs:/usr/local/apache2/logs

volumes:
  apache_html:
  apache_logs:
```

## Custom Apache Configuration

Create `httpd-conf/httpd-vhosts.conf`:

```apache
# Enable virtual hosting

LoadModule vhost_alias_module modules/mod_vhost_alias.so

<VirtualHost *:80>
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com
    DocumentRoot /usr/local/apache2/htdocs/site1

    <Directory /usr/local/apache2/htdocs/site1>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog /usr/local/apache2/logs/site1-error.log
    CustomLog /usr/local/apache2/logs/site1-access.log combined
</VirtualHost>
```

## Enabling SSL/TLS

Apache's main config needs SSL module enabled. Use a custom httpd.conf or extend via Portainer:

```bash
# Via Portainer console - enable SSL module
docker exec apache sh -c 'echo "LoadModule ssl_module modules/mod_ssl.so" >> /usr/local/apache2/conf/httpd.conf'
docker exec apache sh -c 'echo "Include conf/extra/httpd-ssl.conf" >> /usr/local/apache2/conf/httpd.conf'
```

For a simpler SSL approach, use Nginx or Traefik as the SSL terminator in front of Apache.

## Enabling .htaccess (mod_rewrite)

Many PHP apps (WordPress, Laravel) require mod_rewrite:

```bash
# Exec into container to enable modules
docker exec apache sh -c 'sed -i "s/#LoadModule rewrite/LoadModule rewrite/" /usr/local/apache2/conf/httpd.conf'
docker exec apache sh -c 'sed -i "s/#LoadModule deflate/LoadModule deflate/" /usr/local/apache2/conf/httpd.conf'

# Reload Apache
docker exec apache httpd -k graceful
```

Or create a custom Dockerfile:

```dockerfile
FROM httpd:2.4-alpine

# Enable required modules
RUN sed -i 's/#LoadModule rewrite_module/LoadModule rewrite_module/' /usr/local/apache2/conf/httpd.conf
RUN sed -i 's/#LoadModule deflate_module/LoadModule deflate_module/' /usr/local/apache2/conf/httpd.conf
RUN sed -i 's/AllowOverride None/AllowOverride All/' /usr/local/apache2/conf/httpd.conf
```

Build this image and use it in your Portainer stack.

## Apache with PHP (mod_php)

For PHP applications, use the php-apache image:

```yaml
services:
  php-apache:
    image: php:8.2-apache
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./src:/var/www/html
    environment:
      - PHP_MEMORY_LIMIT=256M
```

## Verifying the Deployment

```bash
# Test Apache configuration
docker exec apache httpd -t
# Expected: Syntax OK

# Test HTTP response
curl -I http://localhost
# Expected: HTTP/1.1 200 OK or 403 Forbidden (if no index file)

# View Apache version
docker exec apache httpd -v
```

## Performance Tuning

```apache
# httpd-conf/httpd-perf.conf
# MPM prefork tuning for PHP applications
<IfModule mpm_prefork_module>
    StartServers          5
    MinSpareServers       5
    MaxSpareServers       10
    MaxRequestWorkers     150
    MaxConnectionsPerChild 1000
</IfModule>
```

## Conclusion

Apache deployed via Portainer works well for PHP applications, legacy apps, and scenarios where Apache's .htaccess ecosystem is needed. Use bind mounts for the document root and configuration so changes can be made on the host and picked up with an Apache graceful reload through Portainer's exec feature.
