# How to Create AWS EventBridge Rules with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, EventBridge, Event-Driven, Infrastructure as Code

Description: Learn how to create AWS EventBridge rules with OpenTofu to trigger Lambda functions, SQS queues, and other targets based on scheduled or event-driven patterns.

AWS EventBridge is a serverless event bus that connects AWS services and custom applications. Managing EventBridge rules in OpenTofu ensures your event routing is version-controlled and consistently deployed across environments.

## Provider Configuration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

## Scheduled Rule (Cron)

```hcl
resource "aws_cloudwatch_event_rule" "daily_cleanup" {
  name                = "daily-cleanup"
  description         = "Trigger daily database cleanup at 2 AM UTC"
  schedule_expression = "cron(0 2 * * ? *)"  # 2:00 AM UTC daily
}

resource "aws_cloudwatch_event_target" "cleanup_lambda" {
  rule      = aws_cloudwatch_event_rule.daily_cleanup.name
  target_id = "cleanup-lambda"
  arn       = aws_lambda_function.cleanup.arn
}

resource "aws_lambda_permission" "allow_eventbridge" {
  statement_id  = "AllowExecutionFromEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.cleanup.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.daily_cleanup.arn
}
```

## Rate-Based Schedule

```hcl
resource "aws_cloudwatch_event_rule" "every_5_minutes" {
  name                = "every-5-minutes-health-check"
  description         = "Run health check every 5 minutes"
  schedule_expression = "rate(5 minutes)"
}

resource "aws_cloudwatch_event_target" "health_check" {
  rule      = aws_cloudwatch_event_rule.every_5_minutes.name
  target_id = "health-check-lambda"
  arn       = aws_lambda_function.health_check.arn

  # Pass a constant input to the Lambda
  input = jsonencode({
    action    = "health_check"
    timestamp = "auto"
  })
}
```

## Pattern-Based Rule (AWS Service Events)

```hcl
# Trigger on EC2 state changes

resource "aws_cloudwatch_event_rule" "ec2_state_change" {
  name        = "ec2-state-change"
  description = "Capture EC2 instance state changes"

  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance State-change Notification"]
    detail = {
      state = ["terminated", "stopped"]
    }
  })
}

resource "aws_cloudwatch_event_target" "ec2_change_sqs" {
  rule      = aws_cloudwatch_event_rule.ec2_state_change.name
  target_id = "ec2-state-sqs"
  arn       = aws_sqs_queue.events.arn
}
```

## Custom Application Events

```hcl
# Custom event bus for application events
resource "aws_cloudwatch_event_bus" "app" {
  name = "myapp-events"
}

resource "aws_cloudwatch_event_rule" "order_created" {
  name           = "order-created"
  description    = "Route order created events to fulfillment Lambda"
  event_bus_name = aws_cloudwatch_event_bus.app.name

  event_pattern = jsonencode({
    source      = ["myapp.orders"]
    detail-type = ["OrderCreated"]
  })
}

resource "aws_cloudwatch_event_target" "order_fulfillment" {
  rule           = aws_cloudwatch_event_rule.order_created.name
  event_bus_name = aws_cloudwatch_event_bus.app.name
  target_id      = "fulfillment-lambda"
  arn            = aws_lambda_function.fulfillment.arn

  # Pass only specific fields to the target
  input_transformer {
    input_paths = {
      order_id   = "$.detail.orderId"
      customer   = "$.detail.customerId"
    }
    input_template = "{\"orderId\": \"<order_id>\", \"customerId\": \"<customer>\"}"
  }
}
```

## Multiple Targets for One Rule

```hcl
resource "aws_cloudwatch_event_target" "to_lambda" {
  rule      = aws_cloudwatch_event_rule.order_created.name
  event_bus_name = aws_cloudwatch_event_bus.app.name
  target_id = "lambda"
  arn       = aws_lambda_function.fulfillment.arn
}

resource "aws_cloudwatch_event_target" "to_sqs" {
  rule      = aws_cloudwatch_event_rule.order_created.name
  event_bus_name = aws_cloudwatch_event_bus.app.name
  target_id = "sqs"
  arn       = aws_sqs_queue.orders.arn
}

resource "aws_cloudwatch_event_target" "to_sns" {
  rule      = aws_cloudwatch_event_rule.order_created.name
  event_bus_name = aws_cloudwatch_event_bus.app.name
  target_id = "sns"
  arn       = aws_sns_topic.order_notifications.arn
}
```

## Conclusion

AWS EventBridge rules in OpenTofu give you version-controlled event routing. Use schedule expressions for cron jobs, event patterns for reacting to AWS service events, and custom event buses for application-level events. Always grant Lambda functions permission to be invoked by EventBridge, and use input transformers to reshape event data before sending to targets.
