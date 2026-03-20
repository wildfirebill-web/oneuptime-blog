# How to Set Up AWS SES for Email with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, SES, Email, Infrastructure as Code, DKIM, Transactional Email

Description: Learn how to configure AWS Simple Email Service (SES) using OpenTofu for transactional email with domain verification, DKIM signing, and sending authorization.

---

AWS SES is a cost-effective email sending service for transactional and marketing emails. Setting up SES properly - with domain verification, DKIM, DMARC, and appropriate IAM policies - is essential for deliverability. OpenTofu manages this configuration as code, making domain identity setup repeatable.

## Domain Identity and Verification

```hcl
# main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# Create the SES domain identity
resource "aws_ses_domain_identity" "main" {
  domain = var.email_domain
}

# DKIM configuration for signing outbound emails
resource "aws_ses_domain_dkim" "main" {
  domain = aws_ses_domain_identity.main.domain
}
```

## Creating DNS Records for Verification

```hcl
# dns.tf
# Using Route 53 - add the SES verification TXT record
resource "aws_route53_record" "ses_verification" {
  zone_id = var.route53_zone_id
  name    = "_amazonses.${var.email_domain}"
  type    = "TXT"
  ttl     = 600
  records = [aws_ses_domain_identity.main.verification_token]
}

# Add DKIM CNAME records (3 records required)
resource "aws_route53_record" "dkim" {
  count   = 3
  zone_id = var.route53_zone_id
  name    = "${aws_ses_domain_dkim.main.dkim_tokens[count.index]}._domainkey.${var.email_domain}"
  type    = "CNAME"
  ttl     = 600
  records = ["${aws_ses_domain_dkim.main.dkim_tokens[count.index]}.dkim.amazonses.com"]
}

# Add SPF record to authorize SES to send on behalf of your domain
resource "aws_route53_record" "spf" {
  zone_id = var.route53_zone_id
  name    = var.email_domain
  type    = "TXT"
  ttl     = 300
  records = ["v=spf1 include:amazonses.com ~all"]
}

# Add DMARC record
resource "aws_route53_record" "dmarc" {
  zone_id = var.route53_zone_id
  name    = "_dmarc.${var.email_domain}"
  type    = "TXT"
  ttl     = 300
  records = ["v=DMARC1; p=quarantine; rua=mailto:dmarc@${var.email_domain}; pct=100"]
}

# Wait for domain verification before using SES
resource "aws_ses_domain_identity_verification" "main" {
  domain = aws_ses_domain_identity.main.id

  depends_on = [aws_route53_record.ses_verification]
}
```

## Setting Up Email Templates

```hcl
# templates.tf
resource "aws_ses_template" "welcome" {
  name    = "WelcomeEmail"
  subject = "Welcome to {{company_name}}, {{first_name}}!"

  html = <<-HTML
    <html>
      <body>
        <h1>Welcome, {{first_name}}!</h1>
        <p>Thank you for signing up for {{company_name}}.</p>
        <p><a href="{{verify_link}}">Verify your email address</a> to get started.</p>
      </body>
    </html>
  HTML

  text = "Welcome, {{first_name}}! Please verify your email: {{verify_link}}"
}
```

## IAM Policy for SES Sending

```hcl
# iam.tf
# Policy that allows sending email from a specific address/domain
resource "aws_iam_policy" "ses_sender" {
  name        = "SESSenderPolicy"
  description = "Allow sending email via SES"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ses:SendEmail",
          "ses:SendRawEmail",
          "ses:SendTemplatedEmail",
        ]
        Resource = aws_ses_domain_identity.main.arn
        Condition = {
          StringLike = {
            "ses:FromAddress" = ["*@${var.email_domain}"]
          }
        }
      }
    ]
  })
}

# Bounce and complaint handling with SNS
resource "aws_ses_identity_notification_topic" "bounce" {
  topic_arn                = aws_sns_topic.ses_bounces.arn
  notification_type        = "Bounce"
  identity                 = aws_ses_domain_identity.main.domain
  include_original_headers = false
}

resource "aws_ses_identity_notification_topic" "complaint" {
  topic_arn                = aws_sns_topic.ses_complaints.arn
  notification_type        = "Complaint"
  identity                 = aws_ses_domain_identity.main.domain
  include_original_headers = false
}
```

## Best Practices

- Set up DKIM, SPF, and DMARC - all three are required for good deliverability to major inbox providers.
- Subscribe to bounce and complaint notifications via SNS and handle them in your application to maintain a healthy sending reputation.
- Start sending with a suppression list - SES automatically suppresses addresses that have bounced or complained.
- Request production access (move out of sandbox) before going live - sandbox only allows verified email addresses.
- Monitor your bounce rate (keep below 5%) and complaint rate (keep below 0.1%) to maintain healthy sending reputation.
