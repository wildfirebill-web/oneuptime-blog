# How to Use Sidekiq with Redis in Ruby

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Ruby, Sidekiq, Background Jobs, Queue

Description: Learn how to set up Sidekiq with Redis in a Ruby application to process background jobs reliably, with worker classes and job scheduling examples.

---

## What is Sidekiq?

Sidekiq is the most popular background job processor for Ruby. It uses Redis as its job queue and runs multiple worker threads concurrently to process jobs efficiently. Sidekiq is framework-agnostic and works with plain Ruby, Sinatra, or Rails.

## Prerequisites

- Ruby 2.7 or higher
- A running Redis server
- Bundler

## Installing Sidekiq

Add to your `Gemfile`:

```ruby
gem 'sidekiq', '~> 7.0'
```

Then run:

```bash
bundle install
```

## Configuring the Redis Connection

Create a Sidekiq initializer or configure it at startup:

```ruby
# config/initializers/sidekiq.rb (Rails) or sidekiq.rb (plain Ruby)

require 'sidekiq'

Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end
```

For password-protected Redis:

```ruby
Sidekiq.configure_server do |config|
  config.redis = {
    url:      'redis://127.0.0.1:6379/0',
    password: ENV['REDIS_PASSWORD']
  }
end
```

## Creating a Worker Class

```ruby
# app/workers/email_worker.rb (Rails) or workers/email_worker.rb

require 'sidekiq'

class EmailWorker
  include Sidekiq::Worker

  sidekiq_options queue: 'emails', retry: 3

  def perform(user_id, template_name)
    user = User.find(user_id)
    UserMailer.send_email(user, template_name).deliver_now
    puts "Sent #{template_name} to user #{user_id}"
  end
end
```

## Enqueuing Jobs

```ruby
# Enqueue immediately
EmailWorker.perform_async(42, 'welcome')

# Enqueue with a delay
EmailWorker.perform_in(10.minutes, 42, 'reminder')

# Enqueue at a specific time
EmailWorker.perform_at(Time.now + 1.hour, 42, 'followup')
```

## Running the Sidekiq Process

Start Sidekiq from the command line:

```bash
bundle exec sidekiq
```

Specify the queue and concurrency:

```bash
bundle exec sidekiq -q emails -q default -c 10
```

## Sidekiq Configuration File

Create `config/sidekiq.yml`:

```yaml
:concurrency: 10
:queues:
  - [critical, 3]
  - [emails, 2]
  - [default, 1]
:max_retries: 5
```

Run with config:

```bash
bundle exec sidekiq -C config/sidekiq.yml
```

## Defining Multiple Queues

```ruby
class CriticalWorker
  include Sidekiq::Worker
  sidekiq_options queue: 'critical'

  def perform(task_id)
    puts "Processing critical task #{task_id}"
  end
end

class DefaultWorker
  include Sidekiq::Worker
  sidekiq_options queue: 'default'

  def perform(data)
    puts "Processing: #{data}"
  end
end
```

## Retry Logic and Error Handling

```ruby
class PaymentWorker
  include Sidekiq::Worker

  sidekiq_options queue: 'payments', retry: 5

  sidekiq_retries_exhausted do |msg, ex|
    Rails.logger.error "Payment job failed permanently: #{msg['args']} - #{ex.message}"
    # Notify monitoring system
  end

  def perform(payment_id)
    payment = Payment.find(payment_id)
    payment.process!
  rescue Payment::InsufficientFundsError => e
    # Do not retry - push to dead queue immediately
    raise Sidekiq::Death
  rescue StandardError => e
    Rails.logger.warn "Payment #{payment_id} failed, will retry: #{e.message}"
    raise # Re-raise to trigger Sidekiq retry
  end
end
```

## Monitoring Sidekiq via Redis

Inspect job queues programmatically:

```ruby
require 'sidekiq/api'

# Queue stats
queue = Sidekiq::Queue.new('emails')
puts "Jobs in queue: #{queue.size}"

# Process info
Sidekiq::ProcessSet.new.each do |process|
  puts "Worker: #{process['hostname']} - #{process['busy']} busy threads"
end

# Retry set
Sidekiq::RetrySet.new.each do |job|
  puts "Retry: #{job.klass} - #{job.error_message}"
end
```

## Sidekiq Web Dashboard

Add the web UI to your Rails router:

```ruby
# config/routes.rb
require 'sidekiq/web'

Rails.application.routes.draw do
  mount Sidekiq::Web => '/sidekiq'
end
```

Secure it with authentication:

```ruby
authenticate :admin_user do
  mount Sidekiq::Web => '/sidekiq'
end
```

## Summary

Sidekiq uses Redis as its job store and queue backend - jobs are serialized and pushed into Redis lists, and worker processes continuously pop and execute them. Configure the Redis URL with `Sidekiq.configure_server` and `Sidekiq.configure_client`, create worker classes that include `Sidekiq::Worker`, and enqueue work by calling `MyWorker.perform_async(args)`. Sidekiq's built-in retry logic, multiple queue support, and web dashboard make it the most complete background job solution for Ruby.
