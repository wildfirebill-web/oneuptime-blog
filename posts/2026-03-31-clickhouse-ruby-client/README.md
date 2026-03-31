# How to Use ClickHouse Ruby Client

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Ruby, clickhouse-activerecord, HTTP Interface, Query

Description: Learn how to connect to ClickHouse from Ruby using the clickhouse-activerecord gem or the low-level HTTP interface with net/http.

---

Ruby applications can access ClickHouse via HTTP. The most popular option is the `clickhouse-activerecord` gem, which provides an ActiveRecord adapter. For non-Rails scripts, you can also call the HTTP interface directly.

## Installation

```bash
gem install clickhouse-activerecord
# or in Gemfile
gem 'clickhouse-activerecord', '~> 1.0'
```

## Connecting with ActiveRecord

In a Rails `database.yml`:

```text
development:
  adapter: clickhouse
  host: localhost
  port: 8123
  database: default
  username: default
  password:
```

Or configure programmatically:

```ruby
require 'active_record'
require 'clickhouse-activerecord'

ActiveRecord::Base.establish_connection(
  adapter:  'clickhouse',
  host:     'localhost',
  port:     8123,
  database: 'default',
  username: 'default',
  password: ''
)
```

## Defining a Model

```ruby
class Event < ActiveRecord::Base
  self.table_name = 'events'
end
```

## Querying

```ruby
# All events
Event.all.limit(10).each { |e| puts e.inspect }

# Aggregation
counts = Event.group(:event_type).count
counts.each { |type, cnt| puts "#{type}: #{cnt}" }

# Conditions
Event.where(event_type: 'pageview').where('event_date >= ?', '2024-01-01').count
```

## Inserting

```ruby
Event.create!(id: 1, event_date: Date.today, event_type: 'pageview')

# Bulk insert
Event.insert_all([
  { id: 2, event_date: Date.today, event_type: 'click' },
  { id: 3, event_date: Date.today, event_type: 'purchase' }
])
```

## Raw HTTP with net/http

For scripts without Rails, call the HTTP interface directly:

```ruby
require 'net/http'
require 'uri'
require 'json'

uri = URI('http://localhost:8123/')
uri.query = URI.encode_www_form(query: 'SELECT number FROM numbers(5) FORMAT JSON')

response = Net::HTTP.get_response(uri)
data = JSON.parse(response.body)
data['data'].each { |row| puts row['number'] }
```

## Creating Tables via Migration

```ruby
class CreateEvents < ActiveRecord::Migration[7.0]
  def up
    create_table :events, id: false, options: 'ENGINE = MergeTree() ORDER BY (event_date, id)' do |t|
      t.integer  :id,         null: false
      t.date     :event_date, null: false
      t.string   :event_type, null: false
    end
  end
end
```

## Summary

`clickhouse-activerecord` gives Rails applications a familiar ActiveRecord interface to ClickHouse. Use standard `where`, `group`, `count`, and `insert_all` methods for most workloads. For lightweight Ruby scripts, the raw HTTP interface via `net/http` requires no extra dependencies.
