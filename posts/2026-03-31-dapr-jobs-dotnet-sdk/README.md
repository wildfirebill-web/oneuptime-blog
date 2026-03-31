# How to Use Dapr Jobs with .NET SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, .NET, Job, Scheduler, C#

Description: Schedule and manage Dapr Jobs from .NET applications using the Jobs API for one-time and recurring tasks with persistence and callback handling.

---

## Overview

The Dapr Jobs API (v1.14+) provides a durable scheduler for one-time and recurring jobs. Unlike cron bindings, jobs survive Dapr restarts and can be managed programmatically. The .NET SDK exposes jobs via the gRPC API.

## Prerequisites

```bash
dotnet add package Dapr.Client
```

Dapr v1.14+ with scheduler service enabled.

## Step 1: Schedule a One-Time Job

```csharp
using Dapr.Client;
using Google.Protobuf.WellKnownTypes;

public class JobScheduler
{
    private readonly DaprClient _dapr;

    public JobScheduler(DaprClient dapr) => _dapr = dapr;

    public async Task ScheduleOneTimeJob(string jobName, DateTime runAt, object payload)
    {
        await _dapr.ScheduleJobAlpha1Async(
            jobName,
            new Dapr.Client.ScheduleJobRequest
            {
                DueTime = runAt.ToString("o"),
                Data = Any.Pack(new StringValue { Value = System.Text.Json.JsonSerializer.Serialize(payload) })
            }
        );
        Console.WriteLine($"Job '{jobName}' scheduled for {runAt}");
    }
}
```

## Step 2: Schedule a Recurring Job

```csharp
public async Task ScheduleRecurringJob(string jobName, string schedule)
{
    // Cron expression or @every notation
    await _dapr.ScheduleJobAlpha1Async(
        jobName,
        new Dapr.Client.ScheduleJobRequest
        {
            Schedule = schedule,     // "@every 10m" or "0 9 * * MON-FRI"
            Repeats = 0,             // 0 = unlimited
            Data = Any.Pack(new StringValue { Value = "{\"type\":\"report-generation\"}" })
        }
    );
}
```

## Step 3: Handle Job Callbacks

Register an endpoint that Dapr calls when a job triggers:

```csharp
// Program.cs
var app = builder.Build();
app.MapPost("/job/{jobName}", HandleJob);
app.Run();

async Task HandleJob(string jobName, HttpContext ctx, JobScheduler scheduler)
{
    using var reader = new StreamReader(ctx.Request.Body);
    var body = await reader.ReadToEndAsync();
    Console.WriteLine($"Job triggered: {jobName}, data: {body}");

    switch (jobName)
    {
        case "daily-report":
            await GenerateDailyReport();
            break;
        case "cleanup":
            await RunCleanup();
            break;
    }

    ctx.Response.StatusCode = 200;
}
```

## Step 4: Get Job Status

```csharp
public async Task<Dapr.Client.GetJobResponse> GetJob(string jobName)
{
    var job = await _dapr.GetJobAlpha1Async(jobName);
    Console.WriteLine($"Job: {jobName}, Schedule: {job.Schedule}, DueTime: {job.DueTime}");
    return job;
}
```

## Step 5: Delete a Job

```csharp
public async Task CancelJob(string jobName)
{
    await _dapr.DeleteJobAlpha1Async(jobName);
    Console.WriteLine($"Job '{jobName}' cancelled");
}
```

## Example: Order Reminder System

```csharp
public class OrderReminderService
{
    private readonly DaprClient _dapr;

    public OrderReminderService(DaprClient dapr) => _dapr = dapr;

    public async Task SchedulePaymentReminder(string orderId, DateTime dueAt)
    {
        await _dapr.ScheduleJobAlpha1Async(
            $"payment-reminder-{orderId}",
            new Dapr.Client.ScheduleJobRequest
            {
                DueTime = dueAt.ToString("o"),
                Data = Any.Pack(new StringValue { Value = orderId })
            }
        );
    }

    public async Task CancelReminder(string orderId)
    {
        await _dapr.DeleteJobAlpha1Async($"payment-reminder-{orderId}");
    }
}
```

## Summary

Dapr Jobs in .NET provide a simple API for scheduling durable, persistent tasks. Jobs are stored in the Dapr Scheduler service and survive restarts, unlike in-memory timers. The `ScheduleJobAlpha1Async` method handles both one-time (via `DueTime`) and recurring (via `Schedule`) jobs, with callback delivery to your application's HTTP endpoint when the trigger fires.
