# How to Use Dapr AI Integration with .NET SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Ai, Dotnet, Sdk, Microservice, Integration

Description: Learn how to integrate AI capabilities into your .NET applications using the Dapr AI building block and the official Dapr .NET SDK.

---

## Overview

Dapr's AI building block provides a unified API to interact with large language models (LLMs) and other AI services without coupling your application to a specific provider. The Dapr .NET SDK exposes this functionality through a clean, idiomatic interface that fits naturally into your existing .NET code.

## Installing the Dapr .NET SDK

Add the Dapr AI NuGet packages to your project:

```bash
dotnet add package Dapr.AI
dotnet add package Dapr.Client
```

## Configuring the AI Component

Create a Dapr component file that points to your chosen LLM provider. The example below uses the OpenAI component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: openai-llm
spec:
  type: conversation.openai
  version: v1
  metadata:
    - name: key
      secretKeyRef:
        name: openai-secret
        key: api-key
    - name: model
      value: "gpt-4o"
```

## Sending a Conversation Request

Register the Dapr AI client in your .NET service and send a prompt:

```csharp
using Dapr.AI.Conversation;
using Dapr.AI.Conversation.Extensions;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDaprConversationClient();

var app = builder.Build();

app.MapPost("/ask", async (DaprConversationClient client, AskRequest req) =>
{
    var response = await client.ConverseAsync(
        "openai-llm",
        new[] { new ConversationInput(req.Prompt, ConversationRole.User) });

    return Results.Ok(new { Reply = response.Outputs.First().Result });
});

app.Run();

record AskRequest(string Prompt);
```

## Streaming Responses

For longer responses, use the streaming overload to deliver tokens incrementally to the caller:

```csharp
app.MapGet("/stream", async (DaprConversationClient client, HttpContext ctx) =>
{
    ctx.Response.ContentType = "text/event-stream";
    await foreach (var chunk in client.ConverseStreamAsync(
        "openai-llm",
        new[] { new ConversationInput("Write a short poem about Dapr", ConversationRole.User) }))
    {
        await ctx.Response.WriteAsync($"data: {chunk.Delta}\n\n");
        await ctx.Response.Body.FlushAsync();
    }
});
```

## Switching AI Providers at Runtime

Because the provider is defined in the Dapr component file rather than in application code, you can swap from OpenAI to Azure OpenAI or Anthropic by changing the component YAML and restarting the Dapr sidecar - no code changes required:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: openai-llm
spec:
  type: conversation.azure.openai
  version: v1
  metadata:
    - name: endpoint
      value: "https://my-resource.openai.azure.com/"
    - name: model
      value: "gpt-4o"
```

## Running Locally with Dapr CLI

```bash
dapr run --app-id ai-demo --app-port 5000 --components-path ./components \
  -- dotnet run
```

## Summary

The Dapr AI building block lets .NET developers call LLMs through a single abstraction, making it simple to switch providers without modifying application code. By combining `Dapr.AI` with the standard `DaprConversationClient`, you get streaming and synchronous conversation APIs that integrate seamlessly with ASP.NET Core dependency injection.
