# How to Configure Rails Cache Store with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Ruby On Rails, Caching, Cache Store, Ruby

Description: Configure Ruby on Rails to use Redis as the application cache store using the built-in redis-cache-store for fast, distributed caching across processes.

---

## Introduction

Rails ships with several cache stores including memory, file, and memcache. The built-in `RedisCacheStore` (available since Rails 5.2) connects Rails caching to Redis without additional gems. This guide covers configuration, usage, and common caching patterns in Rails with Redis.

## Installation

Add the `redis` gem to your `Gemfile`:

```ruby
gem "redis", ">= 4.0"
# Optional: for connection pooling
gem "connection_pool"
```

```bash
bundle install
```

## Configuration

In `config/environments/production.rb` (and `development.rb` for local testing):

```ruby
config.cache_store = :redis_cache_store, {
  url: ENV.fetch("REDIS_URL", "redis://localhost:6379/1"),
  expires_in: 1.hour,
  namespace: "myapp_cache",
  connect_timeout:    5,
  read_timeout:       1,
  write_timeout:      1,
  reconnect_attempts: 1,
  error_handler: ->(method:, returning:, exception:) {
    Rails.logger.error "Redis cache error: #{exception.class} - #{exception.message}"
    Sentry.capture_exception(exception, level: :warning) if defined?(Sentry)
  }
}
```

## Using the Cache in Models and Controllers

```ruby
# In a controller
class ProductsController < ApplicationController
  def index
    @products = Rails.cache.fetch("products/all", expires_in: 10.minutes) do
      Product.includes(:category).order(:name).to_a
    end
  end

  def show
    @product = Rails.cache.fetch("products/#{params[:id]}", expires_in: 30.minutes) do
      Product.find(params[:id])
    end
  end
end
```

## Cache-Aside Pattern with Versioning

Use cache versioning to automatically invalidate stale data:

```ruby
class Product < ApplicationRecord
  def cached_details
    Rails.cache.fetch(["product", id, updated_at.to_i], expires_in: 1.hour) do
      {
        id: id,
        name: name,
        price: price,
        category: category.name,
        tags: tags.pluck(:name),
      }
    end
  end

  def self.cached_count_by_category
    Rails.cache.fetch("products/count_by_category", expires_in: 5.minutes) do
      group(:category_id).count
    end
  end
end
```

## Low-Level Cache Operations

```ruby
# Write with TTL
Rails.cache.write("key", "value", expires_in: 1.hour)

# Read with default
value = Rails.cache.read("key") || "default"

# Read multiple
values = Rails.cache.read_multi("key1", "key2", "key3")

# Write multiple
Rails.cache.write_multi(
  "key1" => "value1",
  "key2" => "value2",
)

# Delete
Rails.cache.delete("key")

# Delete matched keys
Rails.cache.delete_matched("products/*")

# Increment/Decrement (atomic)
Rails.cache.increment("page_views/home")
Rails.cache.decrement("available_seats/123")

# Exist?
Rails.cache.exist?("key")
```

## Fragment Caching in Views

In `app/views/products/index.html.erb`:

```erb
<% cache "products/list/v1", expires_in: 10.minutes do %>
  <ul>
    <% @products.each do |product| %>
      <% cache product do %>
        <li><%= product.name %> - $<%= product.price %></li>
      <% end %>
    <% end %>
  </ul>
<% end %>
```

## Cache Invalidation on Model Updates

```ruby
class Product < ApplicationRecord
  after_commit :invalidate_cache

  private

  def invalidate_cache
    Rails.cache.delete("products/#{id}")
    Rails.cache.delete("products/all")
    Rails.cache.delete_matched("products/page/*")
  end
end
```

## Connection Pooling for Concurrency

```ruby
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  pool_size: ENV.fetch("RAILS_MAX_THREADS", 5).to_i,
  pool_timeout: 5,
}
```

## Summary

Rails' built-in `RedisCacheStore` requires only adding the `redis` gem and setting `config.cache_store = :redis_cache_store` with the Redis URL. All standard Rails cache methods - `fetch`, `write`, `read`, `delete`, `delete_matched`, `increment` - work as expected. Fragment caching in views and model cache invalidation via `after_commit` callbacks integrate smoothly, making Redis the most practical production cache store for Rails applications.
