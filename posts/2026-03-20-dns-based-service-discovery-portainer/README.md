# How to Set Up DNS-Based Service Discovery in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, DNS, Service Discovery, Networking, Microservices

Description: Use Docker's built-in DNS resolver and custom DNS configurations in Portainer stacks for reliable service-to-service communication using container and service names.

---

Docker provides automatic DNS-based service discovery for containers on the same network. Services can reference each other by container name or service name without hardcoding IP addresses.

## How Docker DNS Works

Every Docker network has an embedded DNS resolver at `127.0.0.11`. When a container makes a DNS query for another container's name, Docker resolves it to the container's IP on that network.

```bash
# From inside a container, these all work if services are on the same network:
curl http://database:5432     # Resolves to postgres container IP
curl http://redis:6379        # Resolves to redis container IP
curl http://api:8080          # Resolves to api container IP
```

## Same-Network Service Discovery

Services on the same custom Docker network discover each other automatically:

```yaml
version: "3.8"
services:
  api:
    image: myapi:1.2.3
    environment:
      # Use service names as hostnames
      - DATABASE_URL=postgres://myapp:password@database:5432/mydb
      - REDIS_URL=redis://cache:6379
    networks:
      - app-net    # Must be on same network as database and cache

  database:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=myapp
      - POSTGRES_PASSWORD=password
    networks:
      - app-net

  cache:
    image: redis:7-alpine
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

## Aliases for Flexible DNS Names

Assign multiple DNS names to a container using network aliases:

```yaml
services:
  postgres-primary:
    image: postgres:16-alpine
    networks:
      app-net:
        aliases:
          - database           # Other containers can use 'database' as hostname
          - db                 # Short alias
          - postgres           # Standard name
```

## Cross-Stack Service Discovery

For services in different Portainer stacks to discover each other:

```yaml
# Stack A — creates the shared network
networks:
  shared-services:
    name: shared-services-network
    driver: bridge

# Stack B — joins the shared network
networks:
  shared-services:
    external: true
    name: shared-services-network
```

## DNS Aliases for Blue/Green Deployments

Use aliases to enable zero-downtime cutover:

```yaml
services:
  app-v2:
    image: myapp:2.0.0
    networks:
      app-net:
        aliases:
          - app    # Swap this alias from app-v1 to app-v2 during cutover
```

## Custom DNS Resolvers

Override DNS for containers that need external resolution:

```yaml
services:
  webapp:
    image: myapp:1.2.3
    dns:
      - 8.8.8.8           # Google DNS
      - 1.1.1.1           # Cloudflare DNS
    dns_search:
      - internal.example.com   # Search domain for unqualified names
```

## Summary

Docker's built-in DNS resolver provides automatic service discovery for containers on the same network. Use custom networks instead of the default bridge for reliable DNS resolution, network aliases for flexible hostname assignment, and external networks to connect services across Portainer stacks.
