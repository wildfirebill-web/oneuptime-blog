# How to Use MySQL with Ruby's mysql2 Gem

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Ruby, mysql2, Gem, Query

Description: A comprehensive guide to the mysql2 Ruby gem covering connection options, query methods, streaming results, and transaction management.

---

## Overview

The `mysql2` gem is a high-performance, native MySQL driver for Ruby. It wraps the C `libmysqlclient` library and provides a simple, idiomatic Ruby API. It powers the `mysql2` ActiveRecord adapter in Rails and can be used standalone in any Ruby application.

## Installation

```bash
gem install mysql2
```

Or in `Gemfile`:

```ruby
gem 'mysql2', '~> 0.5'
```

## Connection Options

```ruby
require 'mysql2'

client = Mysql2::Client.new(
  host:               'localhost',
  port:               3306,
  username:           'app_user',
  password:           ENV['DB_PASSWORD'],
  database:           'shop',
  encoding:           'utf8mb4',
  collation:          'utf8mb4_unicode_ci',
  connect_timeout:    5,
  read_timeout:       30,
  write_timeout:      30,
  reconnect:          true,
  symbolize_keys:     true  # Returns row keys as symbols
)
```

## Query Options

```ruby
# Returns an array of hashes with string keys (default)
results = client.query("SELECT id, name FROM products LIMIT 10")

# Symbolize keys for easier access
results = client.query("SELECT id, name FROM products", symbolize_keys: true)
results.each { |row| puts row[:name] }

# Return as arrays instead of hashes (faster, less memory)
results = client.query("SELECT id, name FROM products", as: :array)
```

## Streaming Large Result Sets

For large queries, stream results row by row to avoid loading everything into memory:

```ruby
client.query("SELECT * FROM large_table", stream: true, cache_rows: false).each do |row|
  process(row)
end
```

## Prepared Statements

```ruby
statement = client.prepare("SELECT id, name FROM users WHERE email = ?")
result    = statement.execute('alice@example.com')
user      = result.first
puts user['name'] if user
```

## Inserting and Updating

```ruby
name    = client.escape("New Product")
price   = 19.99
client.query("INSERT INTO products (name, price) VALUES ('#{name}', #{price})")
puts "Inserted ID: #{client.last_id}"
puts "Rows affected: #{client.affected_rows}"
```

## Transactions

`mysql2` does not have a built-in transaction DSL. Use raw SQL:

```ruby
client.query("START TRANSACTION")
begin
  client.query("UPDATE stock SET qty = qty - 1 WHERE id = 5")
  client.query("INSERT INTO order_items (product_id) VALUES (5)")
  client.query("COMMIT")
rescue Mysql2::Error => e
  client.query("ROLLBACK")
  raise e
end
```

## Error Handling

```ruby
begin
  client.query("INSERT INTO users (email) VALUES ('duplicate@example.com')")
rescue Mysql2::Error => e
  if e.error_number == 1062
    puts "Duplicate entry - email already exists"
  else
    raise e
  end
end
```

## Summary

The `mysql2` gem provides a fast, flexible API for MySQL queries in Ruby. Use `stream: true` for large result sets to keep memory usage low, use prepared statements for repeated queries, and always escape user input or use parameterized execution. Handle `Mysql2::Error` for database-specific exceptions and check `error_number` to branch on specific MySQL error codes.
