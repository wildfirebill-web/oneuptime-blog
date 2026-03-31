# How to Build Dapr Workflows with .NET SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Workflows, .NET, C#, Orchestration

Description: Learn how to build durable, fault-tolerant workflows using the Dapr Workflow API with the .NET SDK, including activities, error handling, and fan-out patterns.

---

## Overview of Dapr Workflows in .NET

Dapr Workflows provide a durable execution engine for orchestrating long-running business processes. Using the .NET SDK, you define workflows as C# classes that survive process restarts, network failures, and infrastructure issues. The Dapr runtime persists workflow state and replays execution history to resume from where it left off.

## Installing the Dapr .NET SDK

```bash
dotnet add package Dapr.Workflow
dotnet add package Dapr.AspNetCore
```

## Defining a Workflow Activity

Activities are the individual units of work in a workflow. They are plain C# classes that extend `WorkflowActivity`:

```csharp
using Dapr.Workflow;

public class SendEmailActivity : WorkflowActivity<EmailRequest, EmailResult>
{
    private readonly IEmailService _emailService;

    public SendEmailActivity(IEmailService emailService)
    {
        _emailService = emailService;
    }

    public override async Task<EmailResult> RunAsync(WorkflowActivityContext context, EmailRequest input)
    {
        Console.WriteLine($"Sending email to {input.Recipient}");
        bool sent = await _emailService.SendAsync(input.Recipient, input.Subject, input.Body);
        return new EmailResult { Success = sent };
    }
}

public record EmailRequest(string Recipient, string Subject, string Body);
public record EmailResult(bool Success);
```

## Defining a Workflow

Workflows orchestrate activities. Define a workflow by extending `Workflow<TInput, TOutput>`:

```csharp
using Dapr.Workflow;

public class OnboardingWorkflow : Workflow<OnboardingRequest, OnboardingResult>
{
    public override async Task<OnboardingResult> RunAsync(WorkflowContext context, OnboardingRequest input)
    {
        // Step 1: Create the user account
        var accountResult = await context.CallActivityAsync<AccountResult>(
            nameof(CreateAccountActivity),
            new CreateAccountRequest(input.Email, input.Name));

        if (!accountResult.Success)
        {
            return new OnboardingResult(false, "Account creation failed");
        }

        // Step 2: Send welcome email
        await context.CallActivityAsync<EmailResult>(
            nameof(SendEmailActivity),
            new EmailRequest(input.Email, "Welcome!", $"Hi {input.Name}, welcome aboard!"));

        // Step 3: Set up default resources in parallel
        var setupTasks = new List<Task<SetupResult>>
        {
            context.CallActivityAsync<SetupResult>(nameof(SetupStorageActivity), accountResult.AccountId),
            context.CallActivityAsync<SetupResult>(nameof(SetupNotificationsActivity), accountResult.AccountId),
        };

        await Task.WhenAll(setupTasks);

        return new OnboardingResult(true, accountResult.AccountId);
    }
}

public record OnboardingRequest(string Email, string Name);
public record OnboardingResult(bool Success, string Message);
```

## Registering Workflows and Activities

Register workflows and activities with the .NET dependency injection container:

```csharp
using Dapr.Workflow;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDaprWorkflow(options =>
{
    options.RegisterWorkflow<OnboardingWorkflow>();
    options.RegisterActivity<CreateAccountActivity>();
    options.RegisterActivity<SendEmailActivity>();
    options.RegisterActivity<SetupStorageActivity>();
    options.RegisterActivity<SetupNotificationsActivity>();
});

builder.Services.AddControllers().AddDapr();

var app = builder.Build();
app.MapControllers();
app.Run();
```

## Starting and Monitoring a Workflow

Use the `DaprWorkflowClient` to start and query workflow instances:

```csharp
using Dapr.Workflow;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/onboarding")]
public class OnboardingController : ControllerBase
{
    private readonly DaprWorkflowClient _workflowClient;

    public OnboardingController(DaprWorkflowClient workflowClient)
    {
        _workflowClient = workflowClient;
    }

    [HttpPost]
    public async Task<IActionResult> StartOnboarding([FromBody] OnboardingRequest request)
    {
        string instanceId = $"onboarding-{Guid.NewGuid():N}";

        await _workflowClient.ScheduleNewWorkflowAsync(
            name: nameof(OnboardingWorkflow),
            instanceId: instanceId,
            input: request);

        return Ok(new { instanceId });
    }

    [HttpGet("{instanceId}")]
    public async Task<IActionResult> GetStatus(string instanceId)
    {
        var state = await _workflowClient.GetWorkflowStateAsync(instanceId);
        if (state == null) return NotFound();

        return Ok(new
        {
            state.RuntimeStatus,
            state.CreatedAt,
            state.LastUpdatedAt,
            Output = state.ReadOutputAs<OnboardingResult>()
        });
    }
}
```

## Handling Errors and Retries

Configure retry policies for activities to handle transient failures:

```csharp
public override async Task<OnboardingResult> RunAsync(WorkflowContext context, OnboardingRequest input)
{
    var retryOptions = new WorkflowTaskOptions
    {
        RetryPolicy = new WorkflowRetryPolicy(
            maxNumberOfAttempts: 3,
            firstRetryInterval: TimeSpan.FromSeconds(5),
            backoffCoefficient: 2.0,
            maxRetryInterval: TimeSpan.FromMinutes(1))
    };

    try
    {
        var result = await context.CallActivityAsync<AccountResult>(
            nameof(CreateAccountActivity),
            new CreateAccountRequest(input.Email, input.Name),
            retryOptions);

        return new OnboardingResult(true, result.AccountId);
    }
    catch (TaskFailedException ex)
    {
        Console.WriteLine($"Activity failed after retries: {ex.Message}");
        return new OnboardingResult(false, "Failed after retries");
    }
}
```

## Running the Application

```bash
dapr run --app-id onboarding-service \
         --app-port 5000 \
         --dapr-http-port 3500 \
         --resources-path ./components \
         -- dotnet run
```

Test by starting a workflow:

```bash
curl -X POST http://localhost:5000/api/onboarding \
  -H "Content-Type: application/json" \
  -d '{"email": "alice@example.com", "name": "Alice"}'
```

## Summary

Dapr Workflows with the .NET SDK let you build complex, durable business processes using familiar C# patterns. Define workflows as typed orchestrator classes, implement activities as injectable services, and use the `DaprWorkflowClient` to start and monitor instances. Retry policies, parallel fan-out, and structured error handling make it straightforward to build production-grade workflow solutions.
