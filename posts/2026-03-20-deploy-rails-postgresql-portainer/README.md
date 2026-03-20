# How to Deploy a Rails + PostgreSQL Stack via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Ruby on Rails, PostgreSQL, Docker Compose, Web Framework, Ruby

Description: Learn how to deploy a Ruby on Rails application with PostgreSQL via Portainer, including database setup, migrations, and production configuration with Puma.

---

Ruby on Rails with PostgreSQL is a battle-tested web stack. Containerizing it via Portainer enables consistent deployments and simplified database management.

## Compose Stack

```yaml
version: "3.8"

services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: rails_production
      POSTGRES_USER: rails
      POSTGRES_PASSWORD: railspass        # Change this
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rails"]
      interval: 10s
      timeout: 5s
      retries: 5

  web:
    image: ruby:3.3-slim
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://rails:railspass@db:5432/rails_production
      RAILS_ENV: production
      RAILS_MASTER_KEY: changeme-32-char-hex    # From config/master.key
      SECRET_KEY_BASE: changeme-very-long-secret-key-base
    volumes:
      - ./app:/app
    working_dir: /app
    # Install gems, setup DB, and start Puma
    command: >
      bash -c "
        apt-get update -qq && apt-get install -y --no-install-recommends libpq-dev build-essential &&
        bundle install &&
        bundle exec rails db:create db:migrate db:seed &&
        bundle exec puma -C config/puma.rb
      "

volumes:
  postgres_data:
```

## Production Puma Configuration

```ruby
# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY") { 2 }
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

bind "tcp://0.0.0.0:3000"
environment ENV.fetch("RAILS_ENV") { "development" }

on_worker_boot do
  ActiveRecord::Base.establish_connection
end
```

## Database Connection Configuration

```yaml
# config/database.yml
production:
  url: <%= ENV['DATABASE_URL'] %>
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

## Running Migrations and Console

```bash
# Run migrations via Portainer Exec console
bundle exec rails db:migrate

# Open Rails console for debugging
bundle exec rails console
```

## Monitoring

Monitor `http://<host>:3000/up` (Rails 7.1+ built-in health check) with OneUptime. Alert immediately on any non-200 response to detect database or application failures.
