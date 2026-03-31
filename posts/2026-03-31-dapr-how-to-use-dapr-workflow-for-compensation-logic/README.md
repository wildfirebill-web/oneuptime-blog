# How to Use Dapr Workflow for Compensation Logic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflow, Saga Pattern, Compensation, Distributed Transaction

Description: Learn how to implement the Saga compensation pattern using Dapr Workflow to handle failures in distributed transactions by rolling back completed steps.

---

## What Is Compensation Logic in Dapr Workflow

Compensation (also called the Saga pattern) is a technique for handling partial failures in distributed transactions. When a multi-step process fails partway through, you cannot use a traditional database rollback across service boundaries. Instead, you execute compensating actions - undo operations - for each step that already succeeded.

Dapr Workflow makes this straightforward because the workflow engine tracks which steps completed, and you can implement try/catch + compensation in a durable, crash-safe way.

## A Concrete Example - Order Fulfillment Saga

An order involves three services: Inventory, Payment, and Shipping. If Shipping fails after Inventory and Payment succeed, you need to refund the payment and restore inventory.

## Implement the Saga Workflow (.NET)

```csharp
using Dapr.Workflow;

public class OrderFulfillmentWorkflow : Workflow<OrderRequest, OrderResult>
{
    public override async Task<OrderResult> RunAsync(WorkflowContext context, OrderRequest order)
    {
        // Track what was completed so we can compensate
        bool inventoryReserved = false;
        bool paymentCharged = false;

        try
        {
            // Step 1: Reserve inventory
            await context.CallActivityAsync(nameof(ReserveInventoryActivity), order);
            inventoryReserved = true;
            context.SetCustomStatus("Inventory reserved");

            // Step 2: Charge payment
            var paymentResult = await context.CallActivityAsync<PaymentResult>(
                nameof(ChargePaymentActivity), order);
            paymentCharged = true;
            context.SetCustomStatus("Payment charged");

            // Step 3: Create shipment - this might fail
            await context.CallActivityAsync(nameof(CreateShipmentActivity), order);
            context.SetCustomStatus("Shipment created");

            return new OrderResult(true, order.OrderId, "Order fulfilled");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[{context.InstanceId}] Failure at step: {ex.Message}");
            context.SetCustomStatus("Compensating...");

            // Compensate in reverse order
            if (paymentCharged)
            {
                try
                {
                    await context.CallActivityAsync(nameof(RefundPaymentActivity), order);
                    Console.WriteLine($"[{context.InstanceId}] Payment refunded");
                }
                catch (Exception compEx)
                {
                    Console.WriteLine($"[{context.InstanceId}] Refund failed: {compEx.Message}");
                    // Log for manual intervention
                }
            }

            if (inventoryReserved)
            {
                try
                {
                    await context.CallActivityAsync(nameof(ReleaseInventoryActivity), order);
                    Console.WriteLine($"[{context.InstanceId}] Inventory released");
                }
                catch (Exception compEx)
                {
                    Console.WriteLine($"[{context.InstanceId}] Inventory release failed: {compEx.Message}");
                }
            }

            return new OrderResult(false, order.OrderId, $"Order failed and compensated: {ex.Message}");
        }
    }
}
```

## Define the Compensable Activities

Each activity and its compensating counterpart:

```csharp
// Forward activity
public class ReserveInventoryActivity : WorkflowActivity<OrderRequest, bool>
{
    public override async Task<bool> RunAsync(WorkflowActivityContext ctx, OrderRequest order)
    {
        Console.WriteLine($"Reserving {order.Quantity} units for product {order.ProductId}");
        // Call inventory service
        await inventoryService.ReserveAsync(order.ProductId, order.Quantity, order.OrderId);
        return true;
    }
}

// Compensating activity
public class ReleaseInventoryActivity : WorkflowActivity<OrderRequest, bool>
{
    public override async Task<bool> RunAsync(WorkflowActivityContext ctx, OrderRequest order)
    {
        Console.WriteLine($"Releasing inventory reservation for order {order.OrderId}");
        await inventoryService.ReleaseAsync(order.ProductId, order.Quantity, order.OrderId);
        return true;
    }
}

public class ChargePaymentActivity : WorkflowActivity<OrderRequest, PaymentResult>
{
    public override async Task<PaymentResult> RunAsync(WorkflowActivityContext ctx, OrderRequest order)
    {
        Console.WriteLine($"Charging ${order.Amount} for order {order.OrderId}");
        return await paymentService.ChargeAsync(order.CustomerId, order.Amount, order.OrderId);
    }
}

public class RefundPaymentActivity : WorkflowActivity<OrderRequest, bool>
{
    public override async Task<bool> RunAsync(WorkflowActivityContext ctx, OrderRequest order)
    {
        Console.WriteLine($"Refunding ${order.Amount} for order {order.OrderId}");
        await paymentService.RefundAsync(order.OrderId);
        return true;
    }
}
```

## Compensation in Python Workflow

```python
from dapr.ext.workflow import DaprWorkflowContext, WorkflowActivityContext
from typing import Any

def order_fulfillment_workflow(ctx: DaprWorkflowContext, order: dict):
    inventory_reserved = False
    payment_charged = False

    try:
        # Step 1
        yield ctx.call_activity(reserve_inventory, input=order)
        inventory_reserved = True

        # Step 2
        yield ctx.call_activity(charge_payment, input=order)
        payment_charged = True

        # Step 3 - may fail
        yield ctx.call_activity(create_shipment, input=order)

        return {"success": True, "orderId": order["orderId"]}

    except Exception as e:
        ctx.set_custom_status(f"Compensating due to: {e}")

        if payment_charged:
            try:
                yield ctx.call_activity(refund_payment, input=order)
            except Exception:
                pass  # log for manual intervention

        if inventory_reserved:
            try:
                yield ctx.call_activity(release_inventory, input=order)
            except Exception:
                pass

        return {"success": False, "error": str(e)}
```

## Handle Compensation Failures

Sometimes compensating actions also fail. Use a persistent compensation log:

```csharp
catch (Exception ex)
{
    // Record pending compensations durably
    var compensations = new List<CompensationRecord>();
    if (paymentCharged)
        compensations.Add(new CompensationRecord("refund-payment", order.OrderId));
    if (inventoryReserved)
        compensations.Add(new CompensationRecord("release-inventory", order.OrderId));

    await context.CallActivityAsync(nameof(RecordPendingCompensationsActivity), compensations);

    // Now attempt compensations
    foreach (var comp in compensations)
    {
        await context.CallActivityAsync(nameof(ExecuteCompensationActivity), comp);
    }
}
```

## Summary

Dapr Workflow makes the Saga compensation pattern straightforward by providing durable, crash-safe execution of both forward steps and their compensating actions. By tracking which activities succeeded and executing compensations in reverse order within a try/catch block, you can implement distributed transaction rollback across service boundaries without a traditional two-phase commit protocol.
