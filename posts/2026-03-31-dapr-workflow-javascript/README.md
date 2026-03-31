# How to Use Dapr Workflow with JavaScript SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflow, JavaScript, Node.js, Orchestration

Description: Learn how to build durable, fault-tolerant workflows in Node.js using the Dapr Workflow API for orchestrating long-running microservice processes.

---

## Introduction

Dapr Workflow lets you define stateful, durable business processes as code. The JavaScript SDK provides workflow and activity APIs so your Node.js services can orchestrate multi-step processes that survive failures and restarts.

## Installing the SDK

```bash
npm install @dapr/dapr
```

## Defining Activities

Activities are the individual steps within a workflow:

```javascript
const { ActivityContext } = require("@dapr/dapr");

async function validateOrderActivity(ctx, orderId) {
  console.log(`Validating order: ${orderId}`);
  // Validation logic
  if (!orderId || orderId.trim() === "") {
    throw new Error("Invalid order ID");
  }
  return { valid: true, orderId };
}

async function processPaymentActivity(ctx, orderId) {
  console.log(`Processing payment for order: ${orderId}`);
  // Payment logic
  return { success: true, transactionId: `txn-${Date.now()}` };
}

async function shipOrderActivity(ctx, orderId) {
  console.log(`Shipping order: ${orderId}`);
  return { trackingNumber: `TRACK-${orderId}` };
}
```

## Defining the Workflow

```javascript
const { WorkflowContext } = require("@dapr/dapr");

async function orderProcessingWorkflow(ctx, orderId) {
  console.log(`Starting workflow for order: ${orderId}`);

  // Call activities sequentially
  const validation = await ctx.callActivity(validateOrderActivity, orderId);

  if (!validation.valid) {
    return { status: "rejected", orderId };
  }

  const payment = await ctx.callActivity(processPaymentActivity, orderId);

  if (!payment.success) {
    return { status: "payment-failed", orderId };
  }

  const shipping = await ctx.callActivity(shipOrderActivity, orderId);

  return {
    status: "completed",
    orderId,
    transactionId: payment.transactionId,
    trackingNumber: shipping.trackingNumber,
  };
}
```

## Registering and Starting the Runtime

```javascript
const { WorkflowRuntime, DaprWorkflowClient } = require("@dapr/dapr");

async function main() {
  const workflowRuntime = new WorkflowRuntime();

  workflowRuntime
    .registerWorkflow(orderProcessingWorkflow)
    .registerActivity(validateOrderActivity)
    .registerActivity(processPaymentActivity)
    .registerActivity(shipOrderActivity);

  await workflowRuntime.start();
  console.log("Workflow runtime started");
}

main().catch(console.error);
```

## Scheduling and Monitoring Workflows

```javascript
const { DaprWorkflowClient } = require("@dapr/dapr");

const workflowClient = new DaprWorkflowClient();

// Start a workflow instance
const instanceId = await workflowClient.scheduleNewWorkflow(
  orderProcessingWorkflow,
  "order-999"
);
console.log("Workflow started:", instanceId);

// Wait for completion
const state = await workflowClient.waitForWorkflowCompletion(
  instanceId,
  undefined,
  30
);
console.log("Workflow result:", state.serializedOutput);

await workflowClient.stop();
```

## Parallel Activity Execution

Run activities in parallel using `Promise.all`:

```javascript
async function parallelWorkflow(ctx, orderIds) {
  const tasks = orderIds.map((id) =>
    ctx.callActivity(validateOrderActivity, id)
  );
  const results = await ctx.when_all(tasks);
  return results;
}
```

## Summary

Dapr Workflow in the JavaScript SDK lets you define durable, orchestrated processes using plain async functions. Activities represent individual steps, and the workflow orchestrator manages sequencing, retries, and state persistence so your processes survive failures without complex infrastructure.
