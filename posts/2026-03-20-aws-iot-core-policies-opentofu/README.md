# How to Create AWS IoT Core Policies with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, IoT Core, IoT Security, Device Policies, Infrastructure as Code

Description: Learn how to create AWS IoT Core policies to control device permissions for publishing, subscribing, and connecting using OpenTofu.

## Introduction

AWS IoT Core policies control what actions a device (certificate) is authorized to perform — connecting, publishing to topics, and subscribing to topics. Well-scoped policies follow the principle of least privilege. OpenTofu manages IoT policies as code.

## Basic Device Policy

```hcl
resource "aws_iot_policy" "device_basic" {
  name = "${var.app_name}-device-basic-${var.environment}"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "iot:Connect"
        Resource = "arn:aws:iot:${var.region}:${var.account_id}:client/${!iot:ClientId}"
      },
      {
        Effect = "Allow"
        Action = ["iot:Publish", "iot:Receive"]
        Resource = [
          "arn:aws:iot:${var.region}:${var.account_id}:topic/devices/${!iot:ClientId}/*"
        ]
      },
      {
        Effect   = "Allow"
        Action   = "iot:Subscribe"
        Resource = "arn:aws:iot:${var.region}:${var.account_id}:topicfilter/devices/${!iot:ClientId}/*"
      }
    ]
  })
}
```

## Sensor-Specific Policy

Restrict a sensor to only publish telemetry data.

```hcl
resource "aws_iot_policy" "sensor" {
  name = "${var.app_name}-sensor-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # Only allow connection from devices with matching ClientId
        Effect   = "Allow"
        Action   = "iot:Connect"
        Resource = "arn:aws:iot:${var.region}:${var.account_id}:client/${!iot:ClientId}"
        Condition = {
          Bool = {
            "iot:Connection.Thing.IsAttached" = "true"  # must be registered thing
          }
        }
      },
      {
        # Publish only to the device's own telemetry topic
        Effect   = "Allow"
        Action   = "iot:Publish"
        Resource = "arn:aws:iot:${var.region}:${var.account_id}:topic/sensors/${!iot:Connection.Thing.ThingName}/telemetry"
      },
      {
        # Subscribe to device-specific commands topic
        Effect   = "Allow"
        Action   = ["iot:Subscribe", "iot:Receive"]
        Resource = [
          "arn:aws:iot:${var.region}:${var.account_id}:topicfilter/sensors/${!iot:Connection.Thing.ThingName}/commands",
          "arn:aws:iot:${var.region}:${var.account_id}:topic/sensors/${!iot:Connection.Thing.ThingName}/commands"
        ]
      },
      {
        # Allow updating device shadow
        Effect   = "Allow"
        Action   = ["iot:GetThingShadow", "iot:UpdateThingShadow"]
        Resource = "arn:aws:iot:${var.region}:${var.account_id}:thing/${!iot:Connection.Thing.ThingName}"
      }
    ]
  })
}
```

## Fleet Policy for All Devices

A broader policy suitable for all devices in a fleet.

```hcl
resource "aws_iot_policy" "fleet" {
  name = "${var.app_name}-fleet-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "iot:Connect"
        Resource = "*"
      },
      {
        Effect   = "Allow"
        Action   = ["iot:Publish", "iot:Subscribe", "iot:Receive"]
        Resource = "arn:aws:iot:${var.region}:${var.account_id}:*"
      }
    ]
  })
}
```

## Attaching Policy to a Certificate

```hcl
resource "aws_iot_policy_attachment" "sensor_cert" {
  policy = aws_iot_policy.sensor.name
  target = var.device_certificate_arn
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

AWS IoT Core policies provide granular authorization for device connections, topic publishing, and subscriptions. OpenTofu manages policy documents with thing-specific variable substitution (`${!iot:ClientId}`) and certificate attachments — enabling secure, least-privilege IoT fleet management.
