# How to Use Dapr Workflow for Order Processing Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflow, Order Processing, Pipeline, E-Commerce

Description: Build a complete order processing pipeline using Dapr workflows - from validation and payment to fulfillment and notification with compensation on failures.

---

Order processing is a classic workflow use case: it involves multiple sequential steps, each of which can fail, and some of which require compensation (rollback) if a later step fails. Dapr workflows handle this reliably with built-in durability and retry logic.

## Order Processing Pipeline Design

The pipeline consists of:
1. Validate order
2. Reserve inventory
3. Process payment
4. Fulfill order (ship)
5. Send confirmation
6. If any step fails after payment: compensate (refund + release inventory)

## Activities

```python
from dapr.ext.workflow import WorkflowActivityContext

def validate_order(ctx: WorkflowActivityContext, order: dict) -> dict:
    if not order.get("items") or order.get("total", 0) <= 0:
        raise ValueError(f"Invalid order: {order.get('id')}")
    return {**order, "validated_at": "2026-03-31T10:00:00Z"}

def reserve_inventory(ctx: WorkflowActivityContext, order: dict) -> dict:
    # Check and decrement stock for each item
    for item in order["items"]:
        stock = get_stock(item["sku"])
        if stock < item["qty"]:
            raise Exception(f"Insufficient stock for SKU {item['sku']}")
        decrement_stock(item["sku"], item["qty"])
    return {**order, "inventory_reserved": True}

def process_payment(ctx: WorkflowActivityContext, order: dict) -> dict:
    charge_result = charge_card(order["payment_method"], order["total"])
    return {**order, "payment_id": charge_result["id"], "paid": True}

def fulfill_order(ctx: WorkflowActivityContext, order: dict) -> dict:
    shipment = create_shipment(order)
    return {**order, "tracking_number": shipment["tracking"], "shipped": True}

def send_confirmation(ctx: WorkflowActivityContext, order: dict) -> dict:
    send_email(order["customer_email"], "order_confirmation", order)
    return {**order, "confirmation_sent": True}

# Compensation activities
def release_inventory(ctx: WorkflowActivityContext, order: dict) -> None:
    for item in order["items"]:
        increment_stock(item["sku"], item["qty"])
    print(f"Inventory released for order {order['id']}")

def refund_payment(ctx: WorkflowActivityContext, order: dict) -> None:
    if order.get("payment_id"):
        issue_refund(order["payment_id"], order["total"])
    print(f"Payment refunded for order {order['id']}")
```

## The Order Workflow with Compensation

```python
from dapr.ext.workflow import DaprWorkflowContext

def order_processing_workflow(ctx: DaprWorkflowContext, order: dict):
    # Track what's been done for compensation
    compensations = []

    try:
        # Step 1: Validate
        order = yield ctx.call_activity(validate_order, input=order)

        # Step 2: Reserve inventory
        order = yield ctx.call_activity(reserve_inventory, input=order)
        compensations.append("inventory")

        # Step 3: Process payment
        order = yield ctx.call_activity(process_payment, input=order)
        compensations.append("payment")

        # Step 4: Fulfill
        order = yield ctx.call_activity(fulfill_order, input=order)

        # Step 5: Notify
        order = yield ctx.call_activity(send_confirmation, input=order)

        return {"status": "completed", "order_id": order["id"], "tracking": order.get("tracking_number")}

    except Exception as e:
        # Compensate in reverse order
        if "payment" in compensations:
            yield ctx.call_activity(refund_payment, input=order)
        if "inventory" in compensations:
            yield ctx.call_activity(release_inventory, input=order)

        return {"status": "failed", "order_id": order.get("id"), "error": str(e)}
```

## Starting and Monitoring Orders

```python
from dapr.ext.workflow import WorkflowRuntime, DaprWorkflowClient

runtime = WorkflowRuntime()
runtime.register_workflow(order_processing_workflow)
for activity in [validate_order, reserve_inventory, process_payment,
                  fulfill_order, send_confirmation, release_inventory, refund_payment]:
    runtime.register_activity(activity)

with runtime:
    client = DaprWorkflowClient()

    order = {
        "id": "ORD-20260331-001",
        "customer_email": "customer@example.com",
        "payment_method": "pm-abc123",
        "total": 149.99,
        "items": [{"sku": "WIDGET-L", "qty": 2, "price": 74.99}]
    }

    instance_id = client.schedule_new_workflow(
        workflow=order_processing_workflow,
        input=order,
        instance_id=f"order-{order['id']}"
    )

    result = client.wait_for_workflow_completion(instance_id, timeout_in_seconds=120)
    print(f"Order result: {result.serialized_output}")
```

## Adding an Approval Step for High-Value Orders

```python
def order_processing_workflow(ctx: DaprWorkflowContext, order: dict):
    order = yield ctx.call_activity(validate_order, input=order)

    # Require approval for orders over $500
    if order["total"] > 500:
        yield ctx.call_activity(request_manager_approval, input=order)
        decision = yield ctx.wait_for_external_event("order-approved")
        if not decision.get("approved"):
            return {"status": "rejected", "order_id": order["id"]}

    # Continue with normal processing...
```

## Summary

Dapr workflows are an excellent fit for order processing pipelines. The sequential activity chain maps naturally to order steps, and the compensation pattern in exception handlers ensures inventory and payment are always rolled back on failure. Custom instance IDs based on order IDs provide idempotency, preventing duplicate order processing on retries. Add approval gates for high-value orders using external events to implement human-in-the-loop controls.
