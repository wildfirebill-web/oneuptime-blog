# How to Use the Dapr Bot for Automation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Bot, Automation, GitHub, Contribution

Description: Use the Dapr Bot GitHub automation to manage pull requests, trigger CI workflows, assign reviewers, and navigate the Dapr contribution process efficiently.

---

## What is the Dapr Bot?

The Dapr Bot (`@dapr-bot`) is a GitHub automation tool that manages the Dapr project's GitHub workflow. It handles PR labeling, CI triggering, reviewer assignment, and release management. Understanding how to interact with it streamlines the contribution process.

## Common Dapr Bot Commands

Dapr Bot responds to slash commands posted as comments on issues and PRs:

```text
/ok-to-test       - Approve a PR for CI testing (requires maintainer)
/assign           - Assign the issue or PR to yourself
/assign @username - Assign to a specific contributor
/lgtm             - Approve the PR (adds lgtm label)
/approve          - Approve and trigger merge (requires maintainer)
/hold             - Put a PR on hold (prevents auto-merge)
/unhold           - Remove hold
/retest           - Rerun failed CI checks
/close            - Close the issue or PR
/cc @username     - Request review from a specific person
```

## Triggering CI with /ok-to-test

First-time contributors need a maintainer to run `/ok-to-test` before CI executes. This prevents malicious code from running in CI:

```bash
# Comment on your PR (maintainer only):
/ok-to-test

# After this, the bot triggers:
# - Unit tests
# - Integration tests
# - E2E tests
# - Linting and formatting checks
```

## Requesting a Review

```bash
# Request specific reviewer
/cc @dapr/maintainers

# Or request by individual
/cc @yourgithubreviewerhandle
```

## Checking Bot Status

The bot posts status updates as comments. Look for:
- "Tests passed" - all CI checks succeeded
- "lgtm" label added - reviewer approved
- "approved" label - maintainer approved, PR can merge

## Automating Your Own Project with Dapr Bot Patterns

You can implement similar automation in your own Dapr projects using GitHub Actions:

```yaml
# .github/workflows/dapr-ci.yml
name: Dapr Application CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Dapr CLI
        run: |
          wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash
          dapr --version

      - name: Initialize Dapr
        run: dapr init

      - name: Start Redis
        run: docker run -d -p 6379:6379 redis:7-alpine

      - name: Run tests with Dapr sidecar
        run: |
          dapr run \
            --app-id test-app \
            --app-port 8080 \
            --dapr-http-port 3500 \
            -- npm test

      - name: Dapr component validation
        run: |
          dapr components -k  # Validate components (Kubernetes)
```

## Label-Based Workflow

The Dapr project uses labels to track PR status:

```text
needs-review     - Awaiting reviewer
lgtm             - Looks Good To Me (one approval)
approved         - Maintainer approved
do-not-merge     - Block merge
size/XS, S, M, L, XL - PR size labels (auto-assigned by bot)
```

Monitor your PR labels:

```bash
gh pr view 1234 --repo dapr/dapr --json labels
```

## Auto-Assign Reviewers via CODEOWNERS

```bash
# .github/CODEOWNERS in your own project
# Automatically requests review from the right team
/src/state/         @team-a
/src/pubsub/        @team-b
/src/workflow/      @team-c
```

## Summary

The Dapr Bot automates GitHub workflows through slash commands like `/ok-to-test`, `/lgtm`, and `/retest`, allowing contributors and maintainers to manage the PR lifecycle efficiently. For your own Dapr projects, GitHub Actions with the Dapr CLI and CODEOWNERS files provide similar automation to enforce code review workflows and keep CI reliable.
