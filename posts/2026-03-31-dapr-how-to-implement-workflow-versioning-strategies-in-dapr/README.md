# How to Implement Workflow Versioning Strategies in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflow, Versioning, Distributed Systems, Microservices, Durable Execution

Description: Learn strategies for versioning Dapr Workflows to safely evolve long-running workflow definitions without breaking in-flight instances already in progress.

---

Long-running Dapr Workflows can execute for days or weeks. When you need to update the workflow logic - adding a new step, changing retry policies, or reordering activities - in-flight instances are already checkpointed against the old code. Running new code against old history can cause non-determinism errors and crash the orchestrator. This guide covers the core versioning strategies for safely evolving Dapr Workflow definitions in production.

## The Non-Determinism Problem

Dapr Workflow (like Durable Functions and Temporal) replays history to reconstruct state after a process restart. If the workflow code no longer matches the recorded sequence of events, the runtime detects a mismatch and faults the workflow.

Example of a breaking change:

```csharp
// Original version
public override async Task<string> RunAsync(WorkflowContext context, OrderInput input)
{
    await context.CallActivityAsync("ValidateOrder", input);
    await context.CallActivityAsync("ProcessPayment", input);
    return "done";
}

// BROKEN: Inserting a step before ProcessPayment causes non-determinism
// for in-flight instances that already completed ValidateOrder
public override async Task<string> RunAsync(WorkflowContext context, OrderInput input)
{
    await context.CallActivityAsync("ValidateOrder", input);
    await context.CallActivityAsync("CheckFraud", input); // NEW - breaks in-flight instances
    await context.CallActivityAsync("ProcessPayment", input);
    return "done";
}
```

## Strategy 1 - Version Flag with Conditional Branching

Use a version flag passed in the workflow input or stored in state to branch between old and new logic:

```csharp
public record OrderInput(string OrderId, string CustomerId, int Version = 1);

public class OrderWorkflow : Workflow<OrderInput, string>
{
    public override async Task<string> RunAsync(WorkflowContext context, OrderInput input)
    {
        await context.CallActivityAsync<bool>("ValidateOrder", input);

        if (input.Version >= 2)
        {
            // New step added in version 2 - only runs for new instances
            await context.CallActivityAsync<bool>("CheckFraud", input);
        }

        await context.CallActivityAsync<bool>("ProcessPayment", input);

        if (input.Version >= 3)
        {
            // Step added in version 3
            await context.CallActivityAsync<bool>("SendConfirmationEmail", input);
        }

        return $"Order {input.OrderId} completed (v{input.Version})";
    }
}
```

New workflow instances are scheduled with the new version:

```csharp
var order = new OrderInput("order-42", "cust-99", Version: 2);
await client.ScheduleNewWorkflowAsync(nameof(OrderWorkflow), input: order);
```

Old in-flight instances continue replaying with their original version number and skip the new steps safely.

## Strategy 2 - Side-by-Side Workflow Classes

Register multiple workflow classes with different names. Route new instances to the new class while old ones finish on the old class:

```csharp
// V1 - keep until all in-flight instances complete
public class OrderWorkflowV1 : Workflow<OrderInput, string>
{
    public override async Task<string> RunAsync(WorkflowContext context, OrderInput input)
    {
        await context.CallActivityAsync<bool>("ValidateOrder", input);
        await context.CallActivityAsync<bool>("ProcessPayment", input);
        return "done";
    }
}

// V2 - new instances use this
public class OrderWorkflowV2 : Workflow<OrderInput, string>
{
    public override async Task<string> RunAsync(WorkflowContext context, OrderInput input)
    {
        await context.CallActivityAsync<bool>("ValidateOrder", input);
        await context.CallActivityAsync<bool>("CheckFraud", input);
        await context.CallActivityAsync<bool>("ProcessPayment", input);
        await context.CallActivityAsync<bool>("SendConfirmationEmail", input);
        return "done";
    }
}
```

Register both versions:

```csharp
services.AddDaprWorkflow(options =>
{
    options.RegisterWorkflow<OrderWorkflowV1>();
    options.RegisterWorkflow<OrderWorkflowV2>();
    options.RegisterActivity<ValidateOrderActivity>();
    options.RegisterActivity<CheckFraudActivity>();
    options.RegisterActivity<ProcessPaymentActivity>();
    options.RegisterActivity<SendConfirmationEmailActivity>();
});
```

Route based on version logic in your API:

```csharp
[HttpPost("orders")]
public async Task<IActionResult> CreateOrder([FromBody] OrderInput input)
{
    string workflowName = input.Version >= 2
        ? nameof(OrderWorkflowV2)
        : nameof(OrderWorkflowV1);

    await _workflowClient.ScheduleNewWorkflowAsync(
        name: workflowName,
        instanceId: input.OrderId,
        input: input);

    return Accepted(new { instanceId = input.OrderId });
}
```

## Strategy 3 - Draining In-Flight Instances Before Deploying

The safest approach for major rewrites: stop accepting new instances, wait for all in-flight instances to complete, then deploy the new code.

```bash
#!/bin/bash
# drain-workflow.sh

APP_ID="order-processor"
DAPR_PORT=3500

echo "Stopping new workflow intake (deploy feature flag or stop traffic)..."
# Typically done via a load balancer rule or feature flag

echo "Waiting for in-flight workflows to complete..."
while true; do
  # Query running workflows via Dapr HTTP API
  RUNNING=$(curl -s "http://localhost:${DAPR_PORT}/v1.0-alpha1/workflows/dapr/OrderWorkflowV1/query" \
    -H "Content-Type: application/json" \
    -d '{"filter":{"runtimeStatus":["RUNNING","SUSPENDED"]}}' | \
    python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d.get('workflowInstances', [])))")
  
  echo "In-flight instances: ${RUNNING}"
  
  if [ "${RUNNING}" -eq 0 ]; then
    echo "All instances complete. Safe to deploy new version."
    break
  fi
  
  sleep 30
done
```

## Using GetCurrentUtcDateTime for Safe Timing

A common mistake is calling `DateTime.UtcNow` inside a workflow - this is non-deterministic across replays. Always use the Dapr-provided timer:

```csharp
public class OrderWorkflow : Workflow<OrderInput, string>
{
    public override async Task<string> RunAsync(WorkflowContext context, OrderInput input)
    {
        await context.CallActivityAsync<bool>("ValidateOrder", input);

        // WRONG - breaks determinism
        // var now = DateTime.UtcNow;

        // CORRECT - deterministic, replay-safe
        var startTime = context.CurrentUtcDateTime;
        
        await context.CallActivityAsync<bool>("ProcessPayment", input);

        var endTime = context.CurrentUtcDateTime;
        var duration = endTime - startTime;

        return $"Order {input.OrderId} took {duration.TotalSeconds:F1}s";
    }
}
```

## Monitoring In-Flight Workflow Versions

Use the Dapr HTTP API to query and monitor workflows by version:

```bash
# List all running OrderWorkflowV1 instances
curl -X POST \
  "http://localhost:3500/v1.0-alpha1/workflows/dapr/OrderWorkflowV1/query" \
  -H "Content-Type: application/json" \
  -d '{
    "filter": {
      "runtimeStatus": ["RUNNING", "SUSPENDED"]
    },
    "maxInstanceCount": 100
  }'

# Get status of a specific instance
curl "http://localhost:3500/v1.0-alpha1/workflows/dapr/OrderWorkflowV1/{instanceId}"

# Terminate a stuck instance if needed
curl -X POST \
  "http://localhost:3500/v1.0-alpha1/workflows/dapr/OrderWorkflowV1/{instanceId}/terminate"
```

## Summary

Versioning Dapr Workflows safely requires understanding the replay-based execution model. The three primary strategies are: versioned input with conditional branching (works for additive changes), side-by-side workflow class registration (cleanest for larger rewrites), and full drain-before-deploy (most reliable for breaking changes). Always use `context.CurrentUtcDateTime` instead of `DateTime.UtcNow`, and test version compatibility by running the new code against recorded histories before deploying to production.
