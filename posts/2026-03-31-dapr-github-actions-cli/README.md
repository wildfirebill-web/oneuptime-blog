# How to Use GitHub Actions with Dapr CLI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, GitHub Action, CI/CD, DevOps, Automation

Description: Learn how to integrate Dapr CLI into GitHub Actions workflows for testing Dapr applications locally in CI, deploying components, and running integration tests.

---

GitHub Actions provides a powerful environment for CI/CD, and the Dapr CLI integrates seamlessly to enable local testing, component validation, and Kubernetes deployment directly from your workflows.

## Installing Dapr CLI in GitHub Actions

```yaml
name: Dapr CI with GitHub Actions
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test-with-dapr:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Install Dapr CLI
      run: |
        wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash
        dapr --version

    - name: Initialize Dapr
      run: dapr init --runtime-version 1.13.0
```

## Running Dapr Applications in CI

```yaml
    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'

    - name: Install dependencies
      run: pip install -r requirements.txt

    - name: Start subscriber in background
      run: |
        dapr run --app-id subscriber --app-port 5001 \
          --resources-path ./components \
          -- python subscriber.py &
        sleep 3

    - name: Run integration tests
      run: |
        dapr run --app-id test-runner \
          --resources-path ./components \
          -- pytest tests/integration/ -v

    - name: Stop Dapr processes
      if: always()
      run: dapr stop --app-id subscriber
```

## Using the Official Dapr Setup Action

```yaml
    - name: Setup Dapr
      uses: dapr/setup-dapr@v2
      with:
        version: '1.13.0'

    - name: Initialize Dapr
      run: dapr init
```

## Deploying Dapr Components to Kubernetes in CI

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: test-with-dapr
    environment: production
    steps:
    - uses: actions/checkout@v4

    - name: Install Dapr CLI
      run: wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash

    - name: Configure kubectl
      uses: azure/k8s-set-context@v3
      with:
        kubeconfig: ${{ secrets.KUBE_CONFIG }}

    - name: Install Dapr on cluster
      run: dapr init --kubernetes --wait

    - name: Apply Dapr components
      run: kubectl apply -f k8s/components/

    - name: Verify Dapr status
      run: |
        dapr status -k
        dapr components -k
```

## Testing Pub/Sub in CI

```yaml
    - name: Test pub/sub integration
      run: |
        # Start the subscriber
        dapr run --app-id test-subscriber \
          --app-port 5001 \
          --resources-path ./components \
          -- python test_subscriber.py &

        sleep 2

        # Publish a test message
        dapr publish \
          --publish-app-id test-subscriber \
          --pubsub pubsub \
          --topic test-topic \
          --data '{"test": true, "runId": "${{ github.run_id }}"}'

        sleep 2

        # Check that the subscriber processed the message
        cat /tmp/received_messages.json | jq '.[] | select(.runId == "${{ github.run_id }}")'
```

## Summary

GitHub Actions integrates with Dapr CLI through either manual installation or the official `dapr/setup-dapr` action. Use `dapr run` to start services in CI, `dapr publish` for pub/sub testing, and `dapr init --kubernetes` for cluster deployments. Run Dapr services in the background during integration tests and always stop them in the `always()` cleanup step.
