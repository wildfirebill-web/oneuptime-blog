# How to Migrate from Temporal to Dapr Workflow

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflow, Temporal, Migration, Orchestration

Description: Learn how to migrate Temporal workflows to Dapr Workflow to simplify infrastructure, reduce operational overhead, and integrate natively with Dapr's ecosystem.

---

## Temporal vs. Dapr Workflow

Temporal is a powerful workflow engine that requires running separate Temporal server and worker processes, a dedicated database (PostgreSQL or MySQL), and the Temporal UI. Dapr Workflow uses the same durable execution model but runs as a Dapr sidecar feature with no separate server infrastructure required. The state store is whatever you already use with Dapr.

## Concept Mapping

| Temporal | Dapr Workflow |
|----------|---------------|
| Workflow function | Workflow class |
| Activity function | Activity class |
| `workflow.ExecuteActivity` | `context.CallActivityAsync` |
| `workflow.GetVersion` | (replay-safe by default) |
| `workflow.Sleep` | `context.CreateTimer` |
| `workflow.GetSignalChannel` | `context.WaitForExternalEventAsync` |
| Worker | Dapr sidecar + service |
| Temporal Server | Dapr runtime |

## Before: Temporal Go Workflow

```go
// workflows/order.go
package workflows

import (
    "time"
    "go.temporal.io/sdk/workflow"
    "myapp/activities"
)

func OrderWorkflow(ctx workflow.Context, input OrderInput) (OrderResult, error) {
    ao := workflow.ActivityOptions{
        StartToCloseTimeout: 10 * time.Second,
        RetryPolicy: &temporal.RetryPolicy{MaximumAttempts: 3},
    }
    ctx = workflow.WithActivityOptions(ctx, ao)

    var reserved bool
    err := workflow.ExecuteActivity(ctx, activities.ReserveInventory, input).Get(ctx, &reserved)
    if err != nil || !reserved {
        return OrderResult{Status: "OutOfStock"}, err
    }

    var paymentId string
    err = workflow.ExecuteActivity(ctx, activities.ProcessPayment, input).Get(ctx, &paymentId)
    if err != nil {
        _ = workflow.ExecuteActivity(ctx, activities.ReleaseInventory, input.OrderId)
        return OrderResult{Status: "PaymentFailed"}, err
    }

    return OrderResult{Status: "Completed", PaymentId: paymentId}, nil
}
```

## After: Dapr Workflow (Go SDK)

```go
// workflows/order.go
package workflows

import (
    "github.com/dapr/go-sdk/workflow"
)

func OrderWorkflow(ctx *workflow.WorkflowContext) (any, error) {
    var input OrderInput
    if err := ctx.GetInput(&input); err != nil {
        return nil, err
    }

    var reserved bool
    if err := ctx.CallActivity(ReserveInventoryActivity, workflow.ActivityInput(input)).
        Await(&reserved); err != nil {
        return OrderResult{Status: "OutOfStock"}, nil
    }

    if !reserved {
        return OrderResult{Status: "OutOfStock"}, nil
    }

    var paymentId string
    if err := ctx.CallActivity(ProcessPaymentActivity, workflow.ActivityInput(input)).
        Await(&paymentId); err != nil {
        ctx.CallActivity(ReleaseInventoryActivity, workflow.ActivityInput(input.OrderId))
        return OrderResult{Status: "PaymentFailed"}, nil
    }

    return OrderResult{Status: "Completed", PaymentId: paymentId}, nil
}
```

## Activity Implementation

```go
// activities/reserve_inventory.go
func ReserveInventoryActivity(ctx workflow.ActivityContext) (any, error) {
    var input OrderInput
    if err := ctx.GetInput(&input); err != nil {
        return false, err
    }
    reserved, err := inventoryService.Reserve(ctx.Context(), input.Items)
    return reserved, err
}
```

## Registration and Worker

Temporal requires a separate worker process. With Dapr, your application is the worker:

```go
// main.go
func main() {
    w, err := workflow.NewWorker()
    if err != nil {
        log.Fatal(err)
    }

    w.RegisterWorkflow(OrderWorkflow)
    w.RegisterActivity(ReserveInventoryActivity)
    w.RegisterActivity(ProcessPaymentActivity)
    w.RegisterActivity(ReleaseInventoryActivity)

    if err = w.Start(); err != nil {
        log.Fatal(err)
    }
    defer w.Shutdown()

    // Start your HTTP server alongside
    http.ListenAndServe(":8080", router)
}
```

## Starting a Workflow

```go
daprClient, _ := dapr.NewClient()
defer daprClient.Close()

resp, err := daprClient.StartWorkflow(ctx, &dapr.StartWorkflowRequest{
    InstanceID:        "order-" + orderId,
    WorkflowComponent: "dapr",
    WorkflowName:      "OrderWorkflow",
    Input:             inputBytes,
})
```

## Summary

Migrating from Temporal to Dapr Workflow eliminates the need to run a separate Temporal server, database, and UI. The programming model is nearly identical - workflows call activities, sleep with timers, and await external events. The main operational difference is that Dapr Workflow state lives in your existing Dapr state store, requiring no additional infrastructure.
