# How to Use Dapr Workflow with Python SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Python, Workflow, Microservice, Orchestration

Description: Learn how to build durable workflows using the Dapr Python SDK, including activity tasks, child workflows, and error handling patterns.

---

## Introduction

Dapr Workflow enables you to write long-running, durable workflows using ordinary code. The Python SDK provides a clean API for defining workflows and activities. This guide walks through building a practical workflow with the `dapr-ext-workflow` package.

## Prerequisites

Install the required packages:

```bash
pip install dapr dapr-ext-workflow
```

Ensure Dapr is initialized locally:

```bash
dapr init
```

## Defining a Workflow

A workflow is a function decorated with `@wf.workflow`. Activities are the individual steps.

```python
import dapr.ext.workflow as wf

wfr = wf.WorkflowRuntime()

@wfr.workflow(name="order_workflow")
def order_workflow(ctx: wf.DaprWorkflowContext, order_id: str):
    result = yield ctx.call_activity(validate_order, input=order_id)
    if not result:
        return "Order invalid"
    payment = yield ctx.call_activity(process_payment, input=order_id)
    return f"Order {order_id} completed: {payment}"

@wfr.activity(name="validate_order")
def validate_order(ctx: wf.ActivityContext, order_id: str) -> bool:
    print(f"Validating order {order_id}")
    return True

@wfr.activity(name="process_payment")
def process_payment(ctx: wf.ActivityContext, order_id: str) -> str:
    print(f"Processing payment for {order_id}")
    return "paid"
```

## Starting the Workflow Runtime

```python
from dapr.clients import DaprClient

wfr.start()

with DaprClient() as client:
    instance_id = client.start_workflow(
        workflow_component="dapr",
        workflow_name="order_workflow",
        input="ORD-001"
    )
    print(f"Started workflow: {instance_id.instance_id}")
```

## Checking Workflow Status

```python
with DaprClient() as client:
    state = client.get_workflow(
        instance_id="<instance-id>",
        workflow_component="dapr"
    )
    print(f"Status: {state.runtime_status}")
    print(f"Result: {state.serialized_output}")
```

## Handling Errors and Retries

Use `RetryPolicy` to configure retry behavior on activities:

```python
retry = wf.WorkflowActivityOptions(
    retry_policy=wf.RetryPolicy(
        max_number_of_attempts=3,
        first_retry_interval=timedelta(seconds=2),
        backoff_coefficient=2.0
    )
)

@wfr.workflow(name="resilient_workflow")
def resilient_workflow(ctx: wf.DaprWorkflowContext, payload: str):
    result = yield ctx.call_activity(
        flaky_activity,
        input=payload,
        options=retry
    )
    return result
```

## Running the Workflow App

Start your Python workflow app with Dapr sidecar:

```bash
dapr run --app-id workflow-app -- python workflow_app.py
```

## Summary

The Dapr Python SDK makes building durable workflows straightforward. You define workflows and activities as Python functions, start the workflow runtime, and use the DaprClient to manage workflow instances. Retry policies and activity options provide resilience without complex infrastructure.
