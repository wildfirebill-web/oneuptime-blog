# How to Configure Atlantis for Multiple OpenTofu Projects

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Atlantis, Multiple Projects, GitOps, Repository Configuration

Description: Learn how to configure Atlantis to manage multiple OpenTofu projects in a monorepo, with per-project settings, dependencies, and workflow customizations.

## Introduction

Infrastructure monorepos often contain dozens of OpenTofu projects across multiple environments and components. Atlantis supports this through its `atlantis.yaml` configuration file, which defines which directories are projects, when to auto-plan them, and how to handle inter-project dependencies.

## Multi-Project atlantis.yaml

```yaml
# atlantis.yaml

version: 3

# Define common settings that apply to all projects
allowed_regexp_prefixes:
  - environments/

projects:
  # Networking layer - runs first
  - name: dev-networking
    dir: environments/dev/networking
    terraform_version: tofu1.7.0
    autoplan:
      when_modified:
        - "*.tf"
        - "**/*.tf"
      enabled: true

  - name: staging-networking
    dir: environments/staging/networking
    terraform_version: tofu1.7.0
    autoplan:
      when_modified: ["*.tf"]
      enabled: true
    apply_requirements:
      - approved

  - name: prod-networking
    dir: environments/prod/networking
    terraform_version: tofu1.7.0
    autoplan:
      when_modified: ["*.tf"]
      enabled: true
    apply_requirements:
      - approved
      - mergeable
    workflow: prod

  # Application layer - depends on networking
  - name: dev-app
    dir: environments/dev/app
    terraform_version: tofu1.7.0
    autoplan:
      when_modified:
        - "*.tf"
        - "../../modules/**/*.tf"  # Replan when modules change
      enabled: true

  - name: prod-app
    dir: environments/prod/app
    terraform_version: tofu1.7.0
    autoplan:
      when_modified: ["*.tf", "../../modules/**/*.tf"]
      enabled: true
    apply_requirements:
      - approved
    workflow: prod

workflows:
  prod:
    plan:
      steps:
        - run: tfsec . --minimum-severity MEDIUM
        - init
        - plan:
            extra_args: ["-lock-timeout=60s"]
    apply:
      steps:
        - apply
```

## Auto-Detecting Projects

For very large monorepos, use Atlantis's auto-detect mode instead of listing all projects:

```yaml
# atlantis.yaml with auto-detection
version: 3

automerge: false
delete_source_branch_on_merge: false

# Auto-detect all directories with .tf files
# and apply when merged to main
```

```yaml
# repos.yaml - Server side configuration for auto-detection
repos:
  - id: github.com/my-org/infrastructure
    apply_requirements: [approved]
    autodiscover:
      mode: auto
      # Only discover projects in these directories
      ignore:
        - "**/modules/**"
        - "**/.terraform/**"
```

## Project-Level Variables per Environment

```yaml
projects:
  - name: prod-database
    dir: environments/prod/database
    workflow: prod

    # Pass environment-specific terraform variables
    terraform_variables:
      environment: "prod"
      instance_class: "db.r6g.large"
      multi_az: "true"

  - name: dev-database
    dir: environments/dev/database
    terraform_variables:
      environment: "dev"
      instance_class: "db.t3.micro"
      multi_az: "false"
```

## Handling Plan-Apply Dependencies

When project B depends on project A's outputs, document this in comments and use a multi-step PR process:

```bash
# In a PR that changes networking (A) and app (B):

# First, apply networking
# atlantis apply -p prod-networking

# Wait for networking apply to complete, then apply app
# atlantis apply -p prod-app
```

## Splitting Large atlantis.yaml

For very large repos, split per team directory:

```yaml
# teams/platform/atlantis.yaml
version: 3
projects:
  - name: platform-network
    dir: platform/networking
  # ...
```

## Conclusion

Multi-project Atlantis configuration scales from a few projects to hundreds by using the `when_modified` file patterns to trigger only relevant plans, requiring approvals for higher environments, and using custom workflows for production. Keep the `atlantis.yaml` in version control alongside your infrastructure code so project configuration changes go through the same review process as HCL changes.
