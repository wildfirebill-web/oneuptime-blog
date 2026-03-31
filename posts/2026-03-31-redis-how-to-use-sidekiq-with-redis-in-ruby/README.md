# How to Use Sidekiq with Redis in Ruby

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Ruby, Sidekiq, Background Job, Rails, Job Queue

Description: Learn how to use Sidekiq with Redis in Ruby for background job processing, including worker setup, job enqueueing, retries, scheduling, and monitoring.

---

## What Is Sidekiq?

Sidekiq is the most popular background job processor for Ruby, using Redis as its job queue. It provides:

- Multi-threaded job processing (default 10 threads per worker)
- Automatic retries with exponential backoff
- Job scheduling for delayed execution
- Web UI for monitoring and management
- Integration with Rails and standalone Ruby apps

## Installation

```ruby
# Gemfile
gem 'sidekiq', '~> 7.0'
```

```bash
bundle install
```

## Basic Configuration

`config/initializers/sidekiq.rb`:

```ruby
Sidekiq.configure_server do |config|
  config.redis = {
    url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0'),
    size: 10  # Connection pool size for Sidekiq server
  }
end

Sidekiq.configure_client do |config|
  config.redis = {
    url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0'),
    size: 5   # Connection pool size for enqueueing
  }
end
```

`config/sidekiq.yml`:

```yaml
---
:concurrency: 10
:queues:
  - [critical, 3]
  - [default, 2]
  - [low, 1]
:max_retries: 5
```

## Creating a Worker

```ruby
class EmailWorker
  include Sidekiq::Worker

  sidekiq_options(
    queue: 'default',
    retry: 5,
    backtrace: true
  )

  def perform(user_id, email_type, options = {})
    user = User.find(user_id)

    case email_type
    when 'welcome'
      UserMailer.welcome_email(user).deliver_now
    when 'order_shipped'
      order_id = options['order_id']
      UserMailer.order_shipped(user, order_id).deliver_now
    else
      raise ArgumentError, "Unknown email type: #{email_type}"
    end

    Rails.logger.info "Email sent: #{email_type} to #{user.email}"
  end
end
```

## Enqueueing Jobs

```ruby
# Enqueue immediately (default queue)
EmailWorker.perform_async(1001, 'welcome')

# Enqueue with arguments
EmailWorker.perform_async(1001, 'order_shipped', { 'order_id' => 'ORD-123' })

# Enqueue with delay
EmailWorker.perform_in(10.minutes, 1001, 'reminder')

# Enqueue at specific time
EmailWorker.perform_at(Time.now + 1.hour, 1001, 'follow_up')

# Check enqueued
Sidekiq::Queue.new('default').size
```

## Multiple Queues with Priority

```ruby
class ReportWorker
  include Sidekiq::Worker

  sidekiq_options queue: 'low', retry: 3

  def perform(report_type, user_id)
    puts "Generating #{report_type} report for user #{user_id}"
    # Generate report...
  end
end

class AlertWorker
  include Sidekiq::Worker

  sidekiq_options queue: 'critical', retry: 10

  def perform(alert_type, details)
    puts "ALERT: #{alert_type} - #{details}"
    # Send alert...
  end
end

# Enqueue to different queues
ReportWorker.perform_async('monthly', 1001)
AlertWorker.perform_async('security_breach', { 'ip' => '1.2.3.4' })
```

## Retry Configuration and Error Handling

```ruby
class PaymentWorker
  include Sidekiq::Worker

  # Custom retry strategy
  sidekiq_options(
    queue: 'critical',
    retry: 5
  )

  sidekiq_retries_exhausted do |job, ex|
    # Called when all retries are exhausted
    user_id = job['args'][0]
    Rails.logger.error "Payment job exhausted for user #{user_id}: #{ex.message}"

    # Notify the user
    UserMailer.payment_failed_notification(user_id).deliver_later
  end

  def perform(user_id, amount, payment_method_id)
    result = PaymentService.charge(
      user_id: user_id,
      amount: amount,
      payment_method: payment_method_id
    )

    raise "Payment failed: #{result.error}" unless result.success?

    OrderService.complete_order(user_id: user_id, amount: amount)
    EmailWorker.perform_async(user_id, 'payment_confirmed')
  end
end
```

## Batch Jobs with Sidekiq Pro (or Manual Batching)

```ruby
class ProductImportWorker
  include Sidekiq::Worker

  sidekiq_options queue: 'low', retry: 3

  def perform(product_ids)
    product_ids.each do |product_id|
      import_product(product_id)
    end
  end

  private

  def import_product(product_id)
    # Import logic
    puts "Importing product #{product_id}"
  end
end

# Chunk and enqueue
all_ids = (1..10000).to_a
all_ids.each_slice(100) do |chunk|
  ProductImportWorker.perform_async(chunk)
end
```

## Starting Sidekiq

```bash
# Basic start
bundle exec sidekiq

# With specific config
bundle exec sidekiq -C config/sidekiq.yml

# With specific queues
bundle exec sidekiq -q critical -q default -q low

# With concurrency
bundle exec sidekiq -c 20
```

## Monitoring with Sidekiq Web UI

Add to `config/routes.rb`:

```ruby
require 'sidekiq/web'

Rails.application.routes.draw do
  # Mount the Sidekiq web UI
  # Protect with authentication
  authenticate :user, ->(u) { u.admin? } do
    mount Sidekiq::Web => '/sidekiq'
  end
end
```

## Checking Queue Status Programmatically

```ruby
require 'sidekiq/api'

# Queue info
queue = Sidekiq::Queue.new('default')
puts "Queue size: #{queue.size}"
puts "Queue latency: #{queue.latency}s"

# Stats
stats = Sidekiq::Stats.new
puts "Processed: #{stats.processed}"
puts "Failed: #{stats.failed}"
puts "Enqueued: #{stats.enqueued}"
puts "Workers: #{stats.workers_size}"

# Dead jobs (exhausted retries)
dead = Sidekiq::DeadSet.new
puts "Dead jobs: #{dead.size}"

# Retry set
retries = Sidekiq::RetrySet.new
puts "Jobs pending retry: #{retries.size}"
```

## Summary

Sidekiq uses Redis as its job queue backbone, storing job payloads in Redis lists for each queue. Configure Redis connection via `Sidekiq.configure_server` and `configure_client` with appropriate pool sizes, define workers with `include Sidekiq::Worker`, and enqueue with `perform_async`. Use `sidekiq_options` for per-worker queue assignment and retry counts, and `sidekiq_retries_exhausted` to handle permanently failed jobs. The Sidekiq Web UI mounted at a protected route provides real-time visibility into queues, workers, and failed jobs.
