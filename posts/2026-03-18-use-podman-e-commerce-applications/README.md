# How to Use Podman for E-Commerce Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, E-Commerce, Containers, WooCommerce, Magento

Description: Learn how to deploy and manage e-commerce applications like WooCommerce and custom storefronts using Podman with rootless containers, pod networking, and persistent storage.

---

> Podman gives e-commerce teams the ability to run complex multi-service storefronts in rootless containers, reducing security risk while keeping deployment simple and reproducible.

E-commerce applications typically involve multiple interconnected services: a web server, an application runtime, a database, a cache layer, and sometimes a search engine. Podman handles this complexity through pods and custom networks, grouping related containers together while maintaining isolation between services.

This guide covers deploying common e-commerce platforms with Podman, including WooCommerce on WordPress, custom Node.js storefronts, and supporting services like Redis and Elasticsearch.

---

## Why Podman for E-Commerce

E-commerce applications handle sensitive data including payment information, customer addresses, and order histories. Podman's rootless containers reduce the blast radius of any compromise. Without a privileged daemon, there is no escalation path from a container breakout to full host access. Podman pods also let you group your application, database, and cache into a single manageable unit that shares a network namespace.

## Deploying WooCommerce with Podman

WooCommerce runs on WordPress, which needs PHP, a web server, and MySQL. Create a pod that groups these services:

```bash
podman pod create --name ecommerce \
  -p 8080:80 \
  -p 3306:3306

podman run -d --pod ecommerce \
  --name ecom-db \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wpuser \
  -e MYSQL_PASSWORD=wppass \
  -v ecom-db-data:/var/lib/mysql:Z \
  docker.io/library/mariadb:11

podman run -d --pod ecommerce \
  --name ecom-wp \
  -e WORDPRESS_DB_HOST=127.0.0.1 \
  -e WORDPRESS_DB_NAME=wordpress \
  -e WORDPRESS_DB_USER=wpuser \
  -e WORDPRESS_DB_PASSWORD=wppass \
  -v ecom-wp-data:/var/www/html:Z \
  docker.io/library/wordpress:php8.2-apache
```

Since both containers share the pod's network namespace, WordPress connects to MariaDB on `127.0.0.1`. Visit `http://localhost:8080` to complete the WordPress setup wizard, then install the WooCommerce plugin.

## Adding a Redis Cache

E-commerce sites benefit from caching to handle traffic spikes. Add Redis to the pod:

```bash
podman run -d --pod ecommerce \
  --name ecom-redis \
  -v ecom-redis-data:/data:Z \
  docker.io/library/redis:7-alpine \
  redis-server --appendonly yes
```

Install a Redis object cache plugin in WordPress and configure it to connect to `127.0.0.1:6379`.

## Custom Node.js Storefront

For a custom storefront built with Node.js:

```dockerfile
# Dockerfile

FROM docker.io/library/node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
ENV NODE_ENV=production
ENV PORT=3000
EXPOSE 3000
CMD ["node", "server.js"]
```

A sample Express-based storefront entry point:

```javascript
// server.js
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.use(express.json());
app.use(express.static('public'));

app.get('/api/products', async (req, res) => {
  // Fetch products from database
  const products = await db.query('SELECT * FROM products WHERE active = true');
  res.json(products);
});

app.get('/api/products/:id', async (req, res) => {
  const product = await db.query('SELECT * FROM products WHERE id = $1', [req.params.id]);
  res.json(product);
});

app.post('/api/cart', async (req, res) => {
  const { productId, quantity } = req.body;
  // Add to cart logic
  res.json({ success: true });
});

app.listen(port, () => {
  console.log(`Storefront running on port ${port}`);
});
```

Build and run in a pod with PostgreSQL:

```bash
podman pod create --name custom-store -p 3000:3000

podman run -d --pod custom-store \
  --name store-db \
  -e POSTGRES_DB=store \
  -e POSTGRES_USER=storeuser \
  -e POSTGRES_PASSWORD=storepass \
  -v store-db-data:/var/lib/postgresql/data:Z \
  docker.io/library/postgres:16-alpine

podman build -t my-storefront .

podman run -d --pod custom-store \
  --name store-app \
  -e DATABASE_URL=postgresql://storeuser:storepass@127.0.0.1:5432/store \
  my-storefront
```

## Adding Search with Elasticsearch

Product search is critical for e-commerce. Add an Elasticsearch container:

```bash
podman run -d --pod ecommerce \
  --name ecom-search \
  -e "discovery.type=single-node" \
  -e "ES_JAVA_OPTS=-Xms256m -Xmx256m" \
  -v ecom-search-data:/usr/share/elasticsearch/data:Z \
  docker.io/library/elasticsearch:8.12.0
```

## Persistent Storage for Product Data

E-commerce data must survive container restarts. Use named volumes for all stateful services:

```bash
podman volume create ecom-db-data
podman volume create ecom-wp-data
podman volume create ecom-uploads

podman volume inspect ecom-db-data
```

For product images and uploads, bind mount a host directory:

```bash
podman run -d --pod ecommerce \
  --name ecom-wp \
  -v /srv/ecommerce/uploads:/var/www/html/wp-content/uploads:Z \
  -v ecom-wp-data:/var/www/html:Z \
  docker.io/library/wordpress:php8.2-apache
```

## Backup Strategy

Automate database backups with a cron-triggered container:

```bash
podman run --rm --pod ecommerce \
  docker.io/library/mariadb:11 \
  sh -c 'mariadb-dump -h 127.0.0.1 -u root -prootpass wordpress' \
  > /srv/backups/ecom-$(date +%Y%m%d).sql
```

Create a systemd timer to run this daily:

```ini
# ~/.config/containers/systemd/ecom-backup.container
[Container]
Image=docker.io/library/mariadb:11
Pod=ecommerce.pod
Exec=sh -c 'mariadb-dump -h 127.0.0.1 -u root -prootpass wordpress > /backups/ecom-$(date +%%Y%%m%%d).sql'
Volume=/srv/backups:/backups:Z

[Service]
Type=oneshot
```

## Security Considerations

E-commerce applications handle payment data, so security is paramount:

```bash
# Run containers as non-root users
podman run -d --user 1000:1000 --name secure-app my-storefront

# Drop all capabilities and add only what is needed
podman run -d \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --name secure-web \
  nginx:stable-alpine

# Use read-only root filesystem
podman run -d \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /run \
  --name locked-app \
  my-storefront
```

## Scaling with podman-compose

For a complete e-commerce stack, define everything in a compose file:

```yaml
# docker-compose.yml (works with podman-compose)
version: "3.9"
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/store
      - REDIS_URL=redis://cache:6379
    depends_on:
      - db
      - cache
  db:
    image: docker.io/library/postgres:16-alpine
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=store
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
  cache:
    image: docker.io/library/redis:7-alpine
    volumes:
      - cache-data:/data

volumes:
  db-data:
  cache-data:
```

Run with:

```bash
podman-compose up -d
```

## Conclusion

Podman provides e-commerce teams with a secure, rootless container runtime that handles multi-service architectures through pods and custom networks. By grouping your application, database, cache, and search services together, you get the isolation benefits of containers with the simplicity of a single deployment unit. The absence of a privileged daemon adds a meaningful layer of security for applications that handle sensitive customer and payment data.
