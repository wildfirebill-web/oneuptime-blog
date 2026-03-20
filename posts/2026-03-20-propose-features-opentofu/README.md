# How to Propose Features for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Feature Proposals, Open Source, RFC, Community, GitHub

Description: Learn how to propose new features for OpenTofu through GitHub discussions, RFCs, and the community process to maximize the chance of your proposal being accepted.

## Introduction

OpenTofu is community-driven and welcomes feature proposals. However, a well-structured proposal that explains the problem, motivates the solution, and considers alternatives is far more likely to be accepted than a vague request. This guide covers how to write effective feature proposals.

## Before Proposing

```bash
# 1. Search existing issues and discussions to avoid duplicates

# https://github.com/opentofu/opentofu/issues
# https://github.com/opentofu/opentofu/discussions

# 2. Check if the feature is in the roadmap
# https://github.com/opentofu/opentofu/blob/main/ROADMAP.md

# 3. Understand the RFC process for significant changes
# https://github.com/opentofu/opentofu/blob/main/rfc/README.md
```

## Types of Feature Proposals

Small enhancements → GitHub Issue
Medium features → GitHub Discussion → Issue
Large architectural changes → RFC (Request for Comments)

## Writing a GitHub Issue Feature Request

```markdown
## Feature Request: Support for `for_each` on provider blocks

### Problem Statement
When managing resources across multiple AWS accounts, users must define
separate `provider` blocks for each account and manually assign them to
resources using `provider` meta-argument. With 20+ accounts, this becomes
unmaintainable.

### Proposed Solution
Allow `for_each` on `provider` blocks, similar to how it works on resources
and modules:

```hcl
provider "aws" {
  for_each = var.accounts
  alias    = each.key
  assume_role {
    role_arn = each.value.role_arn
  }
}

resource "aws_s3_bucket" "per_account" {
  for_each = var.accounts
  provider = aws[each.key]
  bucket   = "my-bucket-${each.key}"
}
```

### Motivation
- **Scalability**: Managing 50+ provider blocks is error-prone
- **DRY**: Eliminates copy-paste provider configuration
- **GitOps**: New accounts can be added by updating a variable file

### Current Workaround
Using a module with a hardcoded provider assignment per account,
requiring a module instance per account.

### Considered Alternatives
1. **Provider-level loops**: Accepted this would require significant internal changes
2. **Dynamic blocks for providers**: Rejected – dynamic blocks don't work for provider arguments
3. **Separate configurations per account**: Doesn't scale, loses cross-account visibility

### Impact Assessment
- Affects: Provider meta-argument handling, plan/apply graph construction
- Breaking changes: None (additive feature)
- Documentation: Provider documentation would need examples

### Community Interest
This is tracked in the OpenTofu community forum with 47 upvotes:
[link to forum post]
```text

## Writing an RFC for Large Features

For features requiring design review, use the RFC process.

```markdown
# RFC-0042: Provider for_each

## Summary
Add `for_each` support to provider configurations to enable dynamic
provider instances based on input variables.

## Motivation
[Detailed problem statement with user stories]

## Detailed Design
[Technical design with examples, pseudocode, and edge cases]

## Drawbacks
[Honest assessment of downsides and complexity]

## Alternatives
[Other approaches considered and why they were rejected]

## Unresolved Questions
[Open questions that need community or maintainer input]
```

File the RFC at:
```bash
# Fork the repo and add your RFC
cp rfc/TEMPLATE.md rfc/0042-provider-for-each.md
# Edit the file, then open a PR
```

## Following Up on Your Proposal

```markdown
# Checklist for proposal follow-up:
# - Respond to questions within a few days
# - Provide additional context if asked
# - Offer to prototype an implementation
# - Link to related issues or prior art
# - Update the proposal based on feedback
```

## Summary

Effective OpenTofu feature proposals clearly state the problem being solved, provide concrete examples of the proposed syntax or behavior, assess impact and alternatives, and demonstrate community interest. Starting with a well-documented GitHub Discussion or Issue and graduating to an RFC for complex features follows the contribution process that gives proposals the best chance of acceptance.
