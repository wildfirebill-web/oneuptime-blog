# How to Build Dapr Workflows with Python SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflows, Python, Orchestration, Distributed Systems

Description: Learn how to build durable workflows with Dapr using the Python SDK, covering workflow definition, activities, error handling, and starting workflow instances.

---

## Overview of Dapr Workflows in Python

The Dapr Python SDK provides a workflow engine that lets you define long-running, durable orchestrations as regular Python functions. Dapr persists workflow state and can replay execution after failures, making workflows resilient to process crashes and network issues.

## Installing the Dapr Python SDK

```bash
pip install dapr dapr-ext-workflow
```

## Defining Workflow Activities

Activities are the building blocks of a workflow. Each activity is a Python function decorated with `@wfr.activity`:

```python
import dapr.ext.workflow as wf

wfr = wf.WorkflowRuntime()

@wfr.activity(name="process_payment")
def process_payment(ctx: wf.ActivityContext, input: dict) -> dict:
    amount = input.get("amount", 0)
    order_id = input.get("order_id")
    print(f"Processing payment of ${amount} for order {order_id}")

    # Simulate payment processing
    if amount <= 0:
        raise ValueError("Invalid payment amount")

    return {
        "success": True,
        "transaction_id": f"txn-{order_id}-{int(amount * 100)}"
    }

@wfr.activity(name="send_confirmation")
def send_confirmation(ctx: wf.ActivityContext, input: dict) -> dict:
    email = input.get("email")
    order_id = input.get("order_id")
    print(f"Sending confirmation email to {email} for order {order_id}")
    return {"sent": True}

@wfr.activity(name="update_inventory")
def update_inventory(ctx: wf.ActivityContext, input: dict) -> dict:
    sku = input.get("sku")
    quantity = input.get("quantity")
    print(f"Reducing inventory for SKU {sku} by {quantity}")
    return {"updated": True, "remaining": 100 - quantity}
```

## Defining the Workflow

A workflow orchestrates activities. Use `@wfr.workflow` to define the orchestration logic:

```python
import dapr.ext.workflow as wf

@wfr.workflow(name="order_processing_workflow")
def order_processing_workflow(ctx: wf.DaprWorkflowContext, order: dict) -> dict:
    order_id = order.get("order_id")
    print(f"Starting order workflow for {order_id}")

    # Step 1: Process payment
    payment_result = yield ctx.call_activity(
        process_payment,
        input={"order_id": order_id, "amount": order.get("total")}
    )

    if not payment_result.get("success"):
        return {"status": "failed", "reason": "Payment declined"}

    # Step 2: Update inventory and send confirmation in parallel
    inventory_task = ctx.call_activity(
        update_inventory,
        input={"sku": order.get("sku"), "quantity": order.get("quantity")}
    )
    email_task = ctx.call_activity(
        send_confirmation,
        input={"email": order.get("customer_email"), "order_id": order_id}
    )

    # Wait for both to complete
    results = yield wf.when_all([inventory_task, email_task])

    return {
        "status": "completed",
        "transaction_id": payment_result.get("transaction_id"),
        "inventory_updated": results[0].get("updated"),
        "email_sent": results[1].get("sent")
    }
```

## Starting the Workflow Runtime

Register the workflow runtime in your application entry point:

```python
import time
import dapr.ext.workflow as wf
from dapr.clients import DaprClient

def main():
    # Start the workflow runtime
    wfr.start()
    print("Workflow runtime started")

    # Give the runtime time to initialize
    time.sleep(2)

    with DaprClient() as client:
        order = {
            "order_id": "ORD-1001",
            "total": 59.99,
            "sku": "WIDGET-42",
            "quantity": 2,
            "customer_email": "customer@example.com"
        }

        # Start a new workflow instance
        instance_id = client.start_workflow(
            workflow_component="dapr",
            workflow_name="order_processing_workflow",
            input=order,
            instance_id="order-ORD-1001"
        )

        print(f"Started workflow: {instance_id}")

        # Wait for completion
        state = client.wait_for_workflow_completion(
            instance_id=instance_id,
            workflow_component="dapr",
            timeout_in_seconds=30
        )

        print(f"Workflow status: {state.runtime_status}")
        print(f"Output: {state.serialized_output}")

if __name__ == "__main__":
    main()
```

## Checking Workflow Status

```python
from dapr.clients import DaprClient

with DaprClient() as client:
    state = client.get_workflow(
        instance_id="order-ORD-1001",
        workflow_component="dapr"
    )
    print(f"Status: {state.runtime_status}")
    print(f"Created: {state.created_at}")
    print(f"Last updated: {state.last_updated_at}")
```

## Error Handling in Workflows

```python
@wfr.workflow(name="resilient_workflow")
def resilient_workflow(ctx: wf.DaprWorkflowContext, input: dict) -> dict:
    try:
        result = yield ctx.call_activity(
            process_payment,
            input=input,
            retry_policy=wf.WorkflowActivityRetryPolicy(
                max_number_of_attempts=3,
                first_retry_interval=timedelta(seconds=5),
                backoff_coefficient=2.0
            )
        )
        return {"success": True, "result": result}
    except Exception as e:
        print(f"Workflow failed: {e}")
        # Run compensation activity
        yield ctx.call_activity(refund_payment, input=input)
        return {"success": False, "error": str(e)}
```

## Running with Dapr

```bash
dapr run --app-id order-service \
         --app-port 5001 \
         --dapr-http-port 3500 \
         --resources-path ./components \
         -- python main.py
```

## Summary

The Dapr Python SDK makes it straightforward to build durable workflow orchestrations using familiar Python functions and the `yield` pattern. Activities handle individual tasks, workflows compose them into business processes, and the Dapr runtime ensures durability across failures. The SDK supports parallel execution, retries, and error compensation, covering the most common workflow patterns.
