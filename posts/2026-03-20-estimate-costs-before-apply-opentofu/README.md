# How to Estimate Costs Before Applying with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infracost, Cost Estimation, FinOps, CI/CD, Infrastructure as Code

Description: Learn how to estimate infrastructure costs before running tofu apply using Infracost, including local cost checks, CI/CD integration, and cost policy gates that block expensive changes.

---

Applying infrastructure without knowing the cost is like buying a car without checking the price. Infracost integrates with OpenTofu to show cost estimates in pull requests, local terminals, and CI/CD pipelines before a single resource is created.

## Cost Estimation Workflow

```mermaid
graph LR
    A[tofu plan] --> B[Infracost breakdown]
    B --> C[Cost estimate]
    C --> D{Threshold check}
    D -->|Under limit| E[PR approved]
    D -->|Over limit| F[Requires approval]
```

## Local Cost Estimation

```bash
# Install Infracost
brew install infracost  # or use the installer script

# Authenticate
infracost auth login

# Get cost breakdown for current directory
infracost breakdown --path .

# Compare with previous state
infracost diff --path .

# Output formats
infracost breakdown --path . --format table
infracost breakdown --path . --format json --out-file /tmp/estimate.json
infracost breakdown --path . --format html --out-file /tmp/estimate.html
```

## Infracost Configuration File

```yaml
# infracost.yml
version: 0.1

projects:
  - path: environments/dev
    name: dev-environment
    terraform_var_files:
      - environments/dev/terraform.tfvars

  - path: environments/staging
    name: staging-environment
    terraform_var_files:
      - environments/staging/terraform.tfvars

  - path: environments/production
    name: production-environment
    terraform_var_files:
      - environments/production/terraform.tfvars
```

## CI/CD Integration with Cost Gate

```yaml
# .github/workflows/cost-check.yml
name: Cost Check
on:
  pull_request:
    paths: ['**.tf', '**.tfvars']

jobs:
  cost:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write

    steps:
      - uses: actions/checkout@v4

      - name: Setup Infracost
        uses: infracost/actions/setup@v2
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Checkout base branch for comparison
        uses: actions/checkout@v4
        with:
          ref: ${{ github.base_ref }}
          path: base

      - name: Generate base cost
        run: |
          infracost breakdown \
            --path base/environments/production \
            --format json \
            --out-file /tmp/base.json

      - name: Generate PR cost
        run: |
          infracost breakdown \
            --path environments/production \
            --format json \
            --out-file /tmp/pr.json

      - name: Check cost increase threshold
        run: |
          # Get the diff percentage
          DIFF=$(infracost diff \
            --path /tmp/pr.json \
            --compare-to /tmp/base.json \
            --format json | jq '.diffTotalMonthlyCost | tonumber')

          echo "Monthly cost change: $DIFF"

          # Fail if monthly cost increases by more than $500
          if (( $(echo "$DIFF > 500" | bc -l) )); then
            echo "ERROR: Cost increase exceeds $500/month threshold"
            exit 1
          fi

      - name: Post cost comment to PR
        run: |
          infracost diff \
            --path /tmp/pr.json \
            --compare-to /tmp/base.json \
            --format diff | \
          infracost comment github \
            --repo ${{ github.repository }} \
            --pull-request ${{ github.event.pull_request.number }} \
            --github-token ${{ github.token }} \
            --behavior update
```

## Usage-Based Cost Estimates

```hcl
# infracost_usage.yml — define expected usage for better estimates
resource "aws_lambda_function" "processor" {
  # Infracost uses these comments for usage estimates
  # infracost-usage: {"monthly_requests": 10000000, "average_request_duration_ms": 100}
  function_name = "data-processor"
  memory_size   = 512
  timeout       = 30
}
```

## Best Practices

- Run `infracost breakdown` before submitting PRs — it takes seconds and surfaces surprises before reviewers see the code.
- Set cost gate thresholds in CI/CD — block PRs that increase monthly spend by more than a defined threshold without manager approval.
- Use Infracost usage files to estimate variable costs (Lambda invocations, API Gateway calls, data transfer) more accurately.
- Share cost estimates in team Slack channels for high-cost changes — visibility drives accountability.
- Compare costs across environments: `infracost diff --path environments/dev --compare-to environments/production` shows how close staging is to production costs.
