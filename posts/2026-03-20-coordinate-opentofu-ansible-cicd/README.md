# How to Coordinate OpenTofu and Ansible in CI/CD Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Ansible, CI/CD, Pipelines, GitHub Actions, Automation

Description: Learn how to design CI/CD pipelines that coordinate OpenTofu infrastructure changes with Ansible configuration management, handling dependencies, rollbacks, and parallel execution.

## Introduction

Coordinating OpenTofu and Ansible in CI/CD requires thinking carefully about ordering, failure handling, and when to run each tool. Infrastructure changes need to complete before configuration management can proceed, but not every code change touches both layers.

## Detecting Changes to Route Pipeline

```yaml
# .github/workflows/deploy.yml

name: Deploy

on:
  push:
    branches: [main]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      infrastructure: ${{ steps.changes.outputs.infrastructure }}
      ansible: ${{ steps.changes.outputs.ansible }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Detect changes
        id: changes
        run: |
          if git diff --name-only HEAD~1 HEAD | grep -q "^infrastructure/"; then
            echo "infrastructure=true" >> $GITHUB_OUTPUT
          else
            echo "infrastructure=false" >> $GITHUB_OUTPUT
          fi

          if git diff --name-only HEAD~1 HEAD | grep -q "^ansible/"; then
            echo "ansible=true" >> $GITHUB_OUTPUT
          else
            echo "ansible=false" >> $GITHUB_OUTPUT
          fi

  provision:
    needs: detect-changes
    if: needs.detect-changes.outputs.infrastructure == 'true'
    runs-on: ubuntu-latest
    outputs:
      db_endpoint: ${{ steps.outputs.outputs.db_endpoint }}
      web_ips: ${{ steps.outputs.outputs.web_ips }}
    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE }}
          aws-region: us-east-1

      - name: OpenTofu Plan
        run: tofu -chdir=infrastructure plan -out=tfplan

      - name: OpenTofu Apply
        run: tofu -chdir=infrastructure apply tfplan

      - name: Extract outputs
        id: outputs
        run: |
          cd infrastructure
          echo "db_endpoint=$(tofu output -raw db_endpoint)" >> $GITHUB_OUTPUT
          echo "web_ips=$(tofu output -json web_server_ips | jq -c .)" >> $GITHUB_OUTPUT

  configure:
    needs: [detect-changes, provision]
    # Run if ansible changed OR if infrastructure changed (new servers need config)
    if: |
      always() &&
      (needs.detect-changes.outputs.ansible == 'true' ||
       needs.provision.result == 'success')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key

      - name: Generate inventory from OpenTofu state
        run: |
          cd infrastructure
          tofu output -raw ansible_ini_inventory > ../ansible/inventory/hosts.ini

      - name: Run Ansible
        run: |
          cd ansible
          ansible-playbook \
            -i inventory/hosts.ini \
            --private-key ~/.ssh/deploy_key \
            -e "db_endpoint=${{ needs.provision.outputs.db_endpoint || '' }}" \
            playbooks/site.yml
```

## Rollback Strategy

```yaml
  rollback:
    needs: [provision, configure]
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          # Check out the previous version
          ref: ${{ github.event.before }}

      - name: Rollback infrastructure
        if: needs.provision.result == 'failure'
        run: |
          cd infrastructure
          tofu init
          tofu apply -auto-approve  # Apply previous version

      - name: Rollback configuration
        if: needs.configure.result == 'failure'
        run: |
          cd ansible
          ansible-playbook \
            -i inventory/hosts.ini \
            playbooks/rollback.yml
```

## Locking to Prevent Concurrent Deployments

```yaml
      - name: Acquire deployment lock
        uses: softprops/turnstyle@v1
        with:
          poll-interval-seconds: 30
          abort-after-seconds: 600
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Environment Promotion Pipeline

```yaml
# .github/workflows/promote.yml
jobs:
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: |
          tofu -chdir=infrastructure/staging apply -auto-approve
          ansible-playbook -i staging-inventory.ini playbooks/site.yml

  deploy-prod:
    needs: deploy-staging
    environment:
      name: production
      url: https://app.example.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to production
        run: |
          tofu -chdir=infrastructure/prod apply -auto-approve
          ansible-playbook -i prod-inventory.ini playbooks/site.yml
```

## Conclusion

Effective OpenTofu + Ansible CI/CD pipelines detect which layer changed and run only the necessary steps. Infrastructure changes always trigger both OpenTofu and Ansible (new servers need configuration), while pure Ansible changes only trigger the Ansible stage. Use environment protections and deployment locks to prevent concurrent runs from corrupting state, and always pass the current OpenTofu outputs to Ansible rather than using stale inventory files.
