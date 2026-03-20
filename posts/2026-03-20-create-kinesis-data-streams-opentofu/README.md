# How to Create AWS Kinesis Data Streams with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Kinesis, Data Streaming, Infrastructure as Code

Description: Learn how to create AWS Kinesis Data Streams with OpenTofu for real-time data ingestion, processing, and analytics pipelines.

Kinesis Data Streams enables real-time streaming of large volumes of data. Managing streams in OpenTofu ensures capacity (shard count), retention, and security settings are consistently configured and version-controlled.

## Creating a Kinesis Data Stream

```hcl
resource "aws_kinesis_stream" "events" {
  name             = "application-events"
  shard_count      = 4  # Each shard handles 1 MB/s write, 2 MB/s read
  retention_period = 24  # Hours (default: 24, max: 8760)

  shard_level_metrics = [
    "IncomingBytes",
    "IncomingRecords",
    "OutgoingBytes",
    "OutgoingRecords",
    "WriteProvisionedThroughputExceeded",
    "ReadProvisionedThroughputExceeded",
    "IteratorAgeMilliseconds",
  ]

  tags = {
    Environment = "production"
    Team        = "data-platform"
  }
}
```

## On-Demand Capacity Mode

```hcl
# On-demand mode: automatically scales capacity
resource "aws_kinesis_stream" "events_on_demand" {
  name = "events-on-demand"

  stream_mode_details {
    stream_mode = "ON_DEMAND"
  }

  retention_period = 48  # 2 days retention

  tags = {
    Environment = "production"
  }
}
```

## Encryption

```hcl
resource "aws_kinesis_stream" "events_encrypted" {
  name        = "events-encrypted"
  shard_count = 2

  encryption_type = "KMS"
  kms_key_id      = aws_kms_key.kinesis.arn
}
```

## Lambda Consumer

```hcl
resource "aws_lambda_event_source_mapping" "kinesis_processor" {
  event_source_arn  = aws_kinesis_stream.events.arn
  function_name     = aws_lambda_function.stream_processor.arn
  starting_position = "LATEST"  # or TRIM_HORIZON to read from beginning

  batch_size                         = 100
  maximum_batching_window_in_seconds = 5
  parallelization_factor             = 2  # Concurrent Lambda per shard

  bisect_batch_on_function_error = true  # Retry half-batches on failure

  destination_config {
    on_failure {
      destination_arn = aws_sqs_queue.kinesis_dlq.arn
    }
  }
}
```

## Enhanced Fan-Out (Dedicated Consumer)

```hcl
# Enhanced fan-out consumer — gets 2 MB/s per consumer instead of shared 2 MB/s
resource "aws_kinesis_stream_consumer" "analytics" {
  name       = "analytics-consumer"
  stream_arn = aws_kinesis_stream.events.arn
}

# Use the consumer ARN in Lambda trigger for enhanced fan-out
resource "aws_lambda_event_source_mapping" "analytics_lambda" {
  event_source_arn  = aws_kinesis_stream_consumer.analytics.arn  # Consumer ARN, not stream ARN
  function_name     = aws_lambda_function.analytics_processor.arn
  starting_position = "LATEST"
  batch_size        = 100
}
```

## CloudWatch Alarms for Kinesis

```hcl
resource "aws_cloudwatch_metric_alarm" "iterator_age" {
  alarm_name          = "kinesis-high-iterator-age"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "GetRecords.IteratorAgeMilliseconds"
  namespace           = "AWS/Kinesis"
  period              = 60
  statistic           = "Maximum"
  threshold           = 60000  # 1 minute lag
  alarm_description   = "Kinesis consumer is falling behind"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    StreamName = aws_kinesis_stream.events.name
  }
}
```

## IAM Permissions

```hcl
resource "aws_iam_role_policy" "producer" {
  role = aws_iam_role.producer.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["kinesis:PutRecord", "kinesis:PutRecords"]
      Resource = aws_kinesis_stream.events.arn
    }]
  })
}

resource "aws_iam_role_policy" "consumer" {
  role = aws_iam_role.consumer.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = [
        "kinesis:GetRecords",
        "kinesis:GetShardIterator",
        "kinesis:DescribeStream",
        "kinesis:ListStreams",
      ]
      Resource = aws_kinesis_stream.events.arn
    }]
  })
}
```

## Conclusion

Kinesis Data Streams in OpenTofu enable real-time event pipelines with controlled capacity and retention. Use provisioned shards for predictable throughput or on-demand mode for variable workloads. Enable enhanced fan-out for high-throughput consumers, monitor iterator age to detect processing lag, and always configure dead letter queues for Lambda consumers to capture unprocessable records.
