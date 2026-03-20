# How to Manage AWS Savings Plans with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Savings Plans, Cost Optimization, Infrastructure as Code

Description: Learn how to purchase and manage AWS Savings Plans with OpenTofu to reduce compute costs by committing to consistent usage in exchange for discounted rates.

AWS Savings Plans offer up to 72% savings compared to On-Demand pricing in exchange for a 1 or 3-year commitment to a consistent usage amount. Managing Savings Plans purchases in OpenTofu keeps financial commitments version-controlled and reviewable.

## Types of Savings Plans

- **Compute Savings Plans**: Most flexible, apply to any EC2 instance, Fargate, and Lambda across regions and instance families. Up to 66% savings.
- **EC2 Instance Savings Plans**: Highest discount (up to 72%), but tied to a specific instance family and region.
- **SageMaker Savings Plans**: Apply to SageMaker usage.

## Purchasing a Savings Plan

```hcl
resource "aws_savingsplans_plan" "compute" {
  savings_plan_type = "Compute"
  payment_option    = "NoUpfront"  # NoUpfront, PartialUpfront, AllUpfront
  commitment        = "100.00"     # USD per hour commitment

  # Term length
  term_duration_in_seconds = 31536000  # 1 year (31536000s)
  # For 3 years: 94608000

  tags = {
    Purpose     = "EC2 and Fargate compute savings"
    Owner       = "platform-team"
    BudgetCode  = "INFRA-2024"
  }
}
```

## EC2 Instance Savings Plan

```hcl
resource "aws_savingsplans_plan" "ec2" {
  savings_plan_type     = "EC2Instance"
  payment_option        = "PartialUpfront"
  commitment            = "50.00"  # Per hour

  # EC2 Instance plans are region-specific
  # Must be purchased in the correct region

  term_duration_in_seconds = 31536000  # 1 year
}
```

## Understanding Commitment Sizing

Before purchasing, analyze your current On-Demand spend:

```bash
# Use AWS Cost Explorer API (or console) to get recommendations
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS
```

```hcl
# Output the current Savings Plans for reference
data "aws_savingsplans_plans" "existing" {
  filter {
    name   = "state"
    values = ["active"]
  }
}

output "active_savings_plans" {
  value = data.aws_savingsplans_plans.existing.plans[*].commitment
}
```

## Cost Allocation Tags

Tag Savings Plans for cost allocation reporting:

```hcl
resource "aws_savingsplans_plan" "tagged" {
  savings_plan_type        = "Compute"
  payment_option           = "NoUpfront"
  commitment               = "200.00"
  term_duration_in_seconds = 31536000

  tags = {
    CostCenter  = "engineering"
    Team        = "platform"
    Commitment  = "1-year"
    PurchasedBy = "opentofu"
  }
}
```

## Budget Alert for Savings Plan Coverage

```hcl
resource "aws_budgets_budget" "savings_plan_coverage" {
  name         = "savings-plan-coverage"
  budget_type  = "SAVINGS_PLANS_COVERAGE"
  limit_amount = "80"  # Alert if coverage drops below 80%
  limit_unit   = "PERCENTAGE"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "LESS_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["finops@example.com"]
  }
}

resource "aws_budgets_budget" "savings_plan_utilization" {
  name         = "savings-plan-utilization"
  budget_type  = "SAVINGS_PLANS_UTILIZATION"
  limit_amount = "100"
  limit_unit   = "PERCENTAGE"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "LESS_THAN"
    threshold                  = 90  # Alert if utilization drops below 90%
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["finops@example.com"]
  }
}
```

## Conclusion

AWS Savings Plans purchased via OpenTofu give you version-controlled financial commitments. Choose Compute Savings Plans for maximum flexibility across EC2, Fargate, and Lambda; choose EC2 Instance Savings Plans for maximum discount on predictable instance usage. Set budget alerts for coverage and utilization to ensure you're maximizing the value of your commitments.
