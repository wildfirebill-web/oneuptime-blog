# How to Use Redis for Rails Action Cable

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rails, Action Cable, WebSocket, Ruby

Description: Learn how to configure Redis as the Action Cable adapter in Rails to enable real-time WebSocket features across multiple server processes.

---

Action Cable brings WebSockets to Rails, but the default async adapter only works in a single-process environment. For production, you need Redis as the pub/sub backbone so that messages published on one Rails server process reach clients connected to any other process.

## Add the Redis Gem

Ensure the `redis` gem is in your `Gemfile`:

```ruby
# Gemfile
gem 'redis', '~> 5.0'
```

Then install it:

```bash
bundle install
```

## Configure the Action Cable Adapter

Update `config/cable.yml` to use the Redis adapter in production:

```yaml
development:
  adapter: async

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: myapp_production
```

The `channel_prefix` prevents key collisions if multiple apps share the same Redis instance.

## Create a Channel

Generate a channel that broadcasts real-time notifications:

```bash
rails generate channel Notifications
```

Edit `app/channels/notifications_channel.rb`:

```ruby
class NotificationsChannel < ApplicationCable::Channel
  def subscribed
    stream_from "notifications:#{current_user.id}"
  end

  def unsubscribed
    stop_all_streams
  end
end
```

## Broadcast from Anywhere

You can broadcast to a channel from a controller, a background job, or a model callback:

```ruby
# From a controller or job
ActionCable.server.broadcast(
  "notifications:#{user.id}",
  { type: "alert", message: "Your order has shipped!" }
)
```

## Subscribe on the Client

In your JavaScript, subscribe to the channel and handle incoming data:

```javascript
import consumer from "./consumer";

consumer.subscriptions.create("NotificationsChannel", {
  received(data) {
    console.log("Notification received:", data.message);
    showNotification(data.message);
  }
});
```

## Mount Action Cable in Production

In `config/routes.rb`, mount the Action Cable server:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  mount ActionCable.server => '/cable'
end
```

Set the allowed request origins in `config/environments/production.rb`:

```ruby
config.action_cable.url = "wss://yourapp.com/cable"
config.action_cable.allowed_request_origins = ["https://yourapp.com"]
```

## Monitor Redis Connection Health

Action Cable silently drops messages if Redis is unreachable. Monitor your Redis instance and set up alerts. A simple health check using redis-cli:

```bash
redis-cli -u $REDIS_URL ping
# Expected output: PONG
```

Integrate this check into your observability platform (such as OneUptime) to get alerted before users notice missing real-time updates.

## Summary

Redis enables Action Cable to work reliably across multiple Rails server processes by acting as a shared pub/sub message bus. Configure `config/cable.yml` to use the Redis adapter in production, broadcast from jobs or controllers using `ActionCable.server.broadcast`, and monitor your Redis connection health to ensure real-time features stay online.
