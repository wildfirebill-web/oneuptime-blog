# How to Set Up Code Review Workflows for OpenTofu Changes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Code Review, GitOps, CI/CD, Infrastructure as Code, Best Practices

Description: Learn how to set up pull request workflows that automatically run tofu plan and post results as comments, enabling infrastructure changes to be reviewed like application code.

## Introduction

Infrastructure changes deserve the same review rigor as application code — but reviewers need to see the *impact* of a change, not just the configuration diff. Automated PR workflows that post `tofu plan` output as comments give reviewers the information they need to approve confidently.

## GitHub Actions: Plan on PR

```yaml
# .github/workflows/opentofu-plan.yml
name: OpenTofu Plan

on:
  pull_request:
    paths:
      - "environments/**"
      - "modules/**"

permissions:
  id-token: write      # For OIDC authentication
  contents: read
  pull-requests: write # For posting comments

jobs:
  plan:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]
    defaults:
      run:
        working-directory: environments/${{ matrix.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - uses: actions/cache@v4
        with:
          path: ~/.terraform.d/plugin-cache
          key: tofu-${{ hashFiles('**/.terraform.lock.hcl') }}

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - name: OpenTofu Init
        run: tofu init -lockfile=readonly

      - name: OpenTofu Plan
        id: plan
        run: |
          tofu plan -no-color -out=tfplan 2>&1 | tee plan-output.txt
          echo "plan_exit_code=${PIPESTATUS[0]}" >> $GITHUB_OUTPUT

      - name: Post Plan as PR Comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const planOutput = fs.readFileSync('environments/${{ matrix.environment }}/plan-output.txt', 'utf8');
            const truncated = planOutput.length > 60000 ? planOutput.slice(0, 60000) + "\n\n... (truncated)" : planOutput;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## OpenTofu Plan — ${{ matrix.environment }}\n\`\`\`\n${truncated}\n\`\`\``
            });
```

## GitHub Actions: Apply on Merge

```yaml
# .github/workflows/opentofu-apply.yml
name: OpenTofu Apply

on:
  push:
    branches: [main]
    paths:
      - "environments/**"
      - "modules/**"

jobs:
  apply:
    runs-on: ubuntu-latest
    environment: production   # Requires GitHub environment approval
    defaults:
      run:
        working-directory: environments/prod

    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - run: tofu init -lockfile=readonly
      - run: tofu apply -auto-approve -no-color
```

## Branch Protection Rules

Configure branch protection to require plan checks before merge:

1. Require status checks to pass before merging
2. Select `opentofu-plan / plan (prod)` as required
3. Require pull request reviews (minimum 1 approval)
4. Restrict who can push to `main`

## Plan Output Checklist for Reviewers

Document what reviewers should check:

```markdown
## Infrastructure Review Checklist

- [ ] Plan shows only expected changes
- [ ] No unexpected resource destructions (additions and modifications are fine)
- [ ] Database instances are not being replaced (check for `forces replacement`)
- [ ] All new resources have required tags
- [ ] Sensitive values are shown as `(sensitive value)` — not as plaintext
- [ ] The number of resources to add/change/destroy is reasonable
```

## Conclusion

Setting up automated `tofu plan` comments on pull requests transforms infrastructure code review from "does the HCL look right?" to "does the plan look right?" — a much more useful question. Combine plan automation with branch protection rules and a review checklist to create a robust, auditable infrastructure change management process.
