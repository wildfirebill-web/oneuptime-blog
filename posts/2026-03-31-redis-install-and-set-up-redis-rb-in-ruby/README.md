# How to Install and Set Up redis-rb in Ruby

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Ruby, Redis-Rb, Gem, Client Library

Description: Learn how to install the redis-rb gem and configure it to connect to a Redis server in your Ruby application with practical examples.

---

## What is redis-rb?

redis-rb is the official Ruby client library for Redis. It provides a clean, idiomatic Ruby interface to all Redis commands and supports features like connection pooling, pipelines, Pub/Sub, and Sentinel. It is the go-to Redis client for plain Ruby applications, Sinatra, and is also used under the hood by Rails when you configure the Redis cache store.

## Prerequisites

- Ruby 2.6 or higher
- Bundler installed
- A running Redis server

## Installing redis-rb

### Using Bundler (Recommended)

Add to your `Gemfile`:

```ruby
gem 'redis', '~> 5.0'
```

Then run:

```bash
bundle install
```

### Without Bundler

```bash
gem install redis
```

## Connecting to Redis

### Default Connection

```ruby
require 'redis'

redis = Redis.new
redis.set('greeting', 'Hello from redis-rb')
puts redis.get('greeting') # Hello from redis-rb
```

### Custom Host and Port

```ruby
require 'redis'

redis = Redis.new(host: '127.0.0.1', port: 6379)
```

### Connecting with a Password

```ruby
require 'redis'

redis = Redis.new(
  host:     '127.0.0.1',
  port:     6379,
  password: 'your-redis-password'
)
```

### Connecting via URL

```ruby
require 'redis'

redis = Redis.new(url: 'redis://:your-password@127.0.0.1:6379/0')
```

The URL can also come from an environment variable:

```ruby
redis = Redis.new(url: ENV['REDIS_URL'])
```

## Selecting a Database

```ruby
require 'redis'

redis = Redis.new(db: 1) # Use database index 1
```

## Testing the Connection

```ruby
require 'redis'

redis = Redis.new

begin
  response = redis.ping
  puts "Connection OK: #{response}" # PONG
rescue Redis::CannotConnectError => e
  puts "Failed to connect: #{e.message}"
end
```

## Basic Key Operations

```ruby
require 'redis'

redis = Redis.new

# SET and GET
redis.set('user:1:name', 'Alice')
puts redis.get('user:1:name') # Alice

# SET with expiration (seconds)
redis.setex('token:abc123', 3600, 'valid')

# Check TTL
puts redis.ttl('token:abc123') # 3600

# Increment a counter
redis.incr('page:views')
redis.incrby('page:views', 5)
puts redis.get('page:views') # 6

# Delete a key
redis.del('user:1:name')

# Check existence
puts redis.exists?('user:1:name') # false
```

## Working with Hashes

```ruby
require 'redis'

redis = Redis.new

# Store a hash
redis.hset('user:1', 'name', 'Alice', 'email', 'alice@example.com', 'age', '30')

# Get a field
puts redis.hget('user:1', 'name') # Alice

# Get all fields
puts redis.hgetall('user:1').inspect
# {"name"=>"Alice", "email"=>"alice@example.com", "age"=>"30"}
```

## Connection Pooling with connection_pool Gem

For multi-threaded applications (Puma, Sidekiq), use a connection pool:

```bash
bundle add connection_pool
```

```ruby
require 'redis'
require 'connection_pool'

REDIS_POOL = ConnectionPool.new(size: 5, timeout: 5) do
  Redis.new(url: ENV.fetch('REDIS_URL', 'redis://localhost:6379'))
end

# Use the pool
REDIS_POOL.with do |redis|
  redis.set('key', 'value')
  puts redis.get('key')
end
```

## Configuring Timeouts

```ruby
require 'redis'

redis = Redis.new(
  host:             '127.0.0.1',
  port:             6379,
  connect_timeout:  1,   # seconds to wait for connection
  read_timeout:     0.5, # seconds to wait for a response
  write_timeout:    0.5
)
```

## Error Handling

```ruby
require 'redis'

redis = Redis.new

begin
  redis.set('key', 'value')
rescue Redis::CannotConnectError => e
  puts "Cannot connect to Redis: #{e.message}"
rescue Redis::CommandError => e
  puts "Redis command error: #{e.message}"
rescue Redis::TimeoutError => e
  puts "Redis operation timed out: #{e.message}"
end
```

## Verifying the Setup

```ruby
require 'redis'

redis = Redis.new

redis.set('test:setup', 'redis-rb-ok')
val = redis.get('test:setup')

if val == 'redis-rb-ok'
  puts "redis-rb is set up correctly!"
else
  puts "Something went wrong."
end

redis.del('test:setup')
```

## Summary

Installing redis-rb is as simple as adding `gem 'redis'` to your Gemfile and running `bundle install`. The client provides an intuitive interface where method names map directly to Redis commands. For production use in multi-threaded environments like Puma or Sidekiq, add the `connection_pool` gem to manage concurrent connections safely.
