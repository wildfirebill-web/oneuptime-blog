# How to Deploy Paperless-ngx via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Paperless-ngx, Document Management, Docker, Self-Hosting, OCR

Description: Learn how to deploy Paperless-ngx, the document management system with OCR, via Portainer with a full stack including PostgreSQL, Redis, and Gotenberg.

---

Paperless-ngx scans, OCRs, and indexes your documents, making them full-text searchable. It supports automatic tag assignment, correspondents, and document types. The full stack requires several services but Portainer's stack editor makes it manageable.

## Prerequisites

- Portainer running
- At least 2GB RAM (OCR processing is CPU and memory intensive)
- A directory for documents and media

## Compose Stack

```yaml
version: "3.8"

services:
  broker:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: paperless
      POSTGRES_USER: paperless
      POSTGRES_PASSWORD: paperlesspass    # Change this
    volumes:
      - pgdata:/var/lib/postgresql/data

  webserver:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    restart: unless-stopped
    depends_on:
      - db
      - broker
    ports:
      - "8000:8000"
    environment:
      PAPERLESS_REDIS: redis://broker:6379
      PAPERLESS_DBHOST: db
      PAPERLESS_DBPASS: paperlesspass     # Must match db service
      PAPERLESS_OCR_LANGUAGE: eng
      PAPERLESS_SECRET_KEY: changeme-very-long-random-string
      PAPERLESS_URL: http://paperless.example.com:8000
      PAPERLESS_ADMIN_USER: admin
      PAPERLESS_ADMIN_PASSWORD: adminpass # Change this
      PAPERLESS_ADMIN_MAIL: admin@example.com
    volumes:
      - paperless_data:/usr/src/paperless/data
      - paperless_media:/usr/src/paperless/media
      - /mnt/paperless/export:/usr/src/paperless/export
      - /mnt/paperless/consume:/usr/src/paperless/consume  # Drop PDFs here

volumes:
  redis_data:
  pgdata:
  paperless_data:
  paperless_media:
```

## Deploying

1. In Portainer go to **Stacks > Add Stack**.
2. Name it `paperless`.
3. Update passwords and create the host directories for `export` and `consume`.
4. Click **Deploy the stack**.

Open `http://<host>:8000` and log in with the admin credentials.

## Consuming Documents

Drop any PDF, image, or document into the `/mnt/paperless/consume` directory. Paperless-ngx will automatically detect, OCR, classify, and index it within a few minutes.

## Monitoring

Use OneUptime to monitor `http://<host>:8000` for HTTP 200. Paperless-ngx downtime means document ingestion stops. Configure an alert so paper documents are not lost if the container crashes.
