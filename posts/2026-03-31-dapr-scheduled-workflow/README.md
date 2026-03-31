# How to Implement Scheduled Workflow with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflow, Scheduler, Cron, Job

Description: Learn how to implement scheduled workflows with Dapr Jobs API to run recurring and time-based business processes without external cron infrastructure.

---

## Scheduled Workflows vs. Cron Jobs

A traditional cron job fires and forgets. If the process crashes mid-run, no retry happens and state is lost. Dapr's approach to scheduling combines the Dapr Jobs API (for triggering) with Dapr Workflow (for durable execution), giving you the best of both worlds.

## Defining the Workflow

```csharp
[DaprWorkflow]
public class DailyReportWorkflow : Workflow<ReportInput, ReportResult>
{
    public override async Task<ReportResult> RunAsync(
        WorkflowContext context, ReportInput input)
    {
        // Gather data
        var salesData = await context.CallActivityAsync<SalesData>(
            nameof(FetchSalesDataActivity),
            new DateRange(input.Date.AddDays(-1), input.Date));

        var inventoryData = await context.CallActivityAsync<InventoryData>(
            nameof(FetchInventoryDataActivity), input.Date);

        // Generate report
        var report = await context.CallActivityAsync<Report>(
            nameof(GenerateReportActivity),
            new ReportInputData(salesData, inventoryData));

        // Distribute
        await context.CallActivityAsync(
            nameof(EmailReportActivity), report);

        await context.CallActivityAsync(
            nameof(ArchiveReportActivity), report);

        return new ReportResult
        {
            ReportId = report.Id,
            GeneratedAt = context.CurrentUtcDateTime
        };
    }
}
```

## Registering a Scheduled Job

Use the Dapr Jobs API to schedule workflow execution. The job fires an HTTP callback to your service endpoint.

```yaml
# jobs/daily-report.yaml
apiVersion: dapr.io/v1alpha1
kind: Job
metadata:
  name: daily-report-job
spec:
  schedule: "0 6 * * *"   # every day at 06:00 UTC
  repeats: 0               # 0 = repeat forever
  data:
    "@type": "type.googleapis.com/google.protobuf.StringValue"
    value: '{"reportType":"daily"}'
```

Apply it:

```bash
dapr job create --job-file jobs/daily-report.yaml
```

## Handling the Job Trigger Endpoint

```csharp
[HttpPost("/jobs/daily-report")]
public async Task<IActionResult> TriggerDailyReport([FromBody] JobPayload payload)
{
    var instanceId = $"daily-report-{DateTime.UtcNow:yyyyMMdd}";

    // Check if already running (idempotency guard)
    var existing = await _daprClient.GetWorkflowAsync(
        instanceId, "dapr", nameof(DailyReportWorkflow));

    if (existing?.RuntimeStatus is "Running" or "Pending")
        return Conflict(new { message = "Report already in progress" });

    await _daprClient.StartWorkflowAsync(
        workflowComponent: "dapr",
        workflowName: nameof(DailyReportWorkflow),
        instanceId: instanceId,
        input: new ReportInput { Date = DateTime.UtcNow.Date });

    return Accepted(new { instanceId });
}
```

## Scheduling a One-Time Future Job

For one-time scheduled tasks, use a due-time instead of a schedule:

```bash
dapr job create \
  --name end-of-quarter-close \
  --due-time "2026-03-31T23:59:00Z" \
  --data '{"period":"Q1-2026"}'
```

## Listing and Deleting Jobs

```bash
# List all jobs
dapr job list

# Delete a job
dapr job delete --name daily-report-job
```

## Summary

Dapr's scheduled workflow pattern combines the Jobs API for reliable triggering with Dapr Workflow for durable, retryable execution. Unlike plain cron jobs, this approach survives process crashes and provides observability through Dapr's workflow state API. Use idempotency guards in your trigger endpoint to prevent duplicate workflow instances when jobs fire more than once.
