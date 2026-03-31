# How to Use Redis as SignalR Backplane in ASP.NET Core

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Asp.Net Core, SignalR, Backplane, Real-Time, C#

Description: Configure Redis as the SignalR backplane in ASP.NET Core to synchronize real-time messages across multiple server instances in a load-balanced deployment.

---

## Introduction

SignalR in a single-server deployment stores hub connections in memory. When you scale out to multiple servers, a client connected to server A cannot receive messages sent from server B. A Redis backplane solves this by having all servers publish and subscribe to a shared Redis channel, so messages sent to any hub are relayed to all servers and their connected clients.

## Installation

```bash
dotnet add package Microsoft.AspNetCore.SignalR.StackExchangeRedis
```

## Configuration in Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSignalR()
    .AddStackExchangeRedis(
        builder.Configuration.GetConnectionString("Redis") ?? "localhost:6379",
        options =>
        {
            options.Configuration.ChannelPrefix = RedisChannel.Literal("myapp");
        }
    );

builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
        policy.AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials()
              .WithOrigins("http://localhost:3000"));
});

var app = builder.Build();
app.UseCors();
app.MapHub<ChatHub>("/chatHub");
app.Run();
```

## appsettings.json

```json
{
  "ConnectionStrings": {
    "Redis": "localhost:6379,abortConnect=false,connectTimeout=5000"
  }
}
```

## Creating a SignalR Hub

```csharp
using Microsoft.AspNetCore.SignalR;

public class ChatHub : Hub
{
    public async Task SendMessage(string room, string user, string message)
    {
        // This is broadcast to ALL servers via Redis backplane
        await Clients.Group(room).SendAsync("ReceiveMessage", new
        {
            user,
            message,
            timestamp = DateTime.UtcNow
        });
    }

    public async Task JoinRoom(string room)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, room);
        await Clients.Group(room).SendAsync("UserJoined", Context.ConnectionId);
    }

    public async Task LeaveRoom(string room)
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, room);
        await Clients.Group(room).SendAsync("UserLeft", Context.ConnectionId);
    }

    public override async Task OnConnectedAsync()
    {
        await base.OnConnectedAsync();
        Console.WriteLine($"Client connected: {Context.ConnectionId}");
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        Console.WriteLine($"Client disconnected: {Context.ConnectionId}");
        await base.OnDisconnectedAsync(exception);
    }
}
```

## Sending Messages from Outside the Hub

Inject `IHubContext` to send messages from controllers or background services:

```csharp
using Microsoft.AspNetCore.SignalR;

[ApiController]
[Route("api/[controller]")]
public class NotificationsController : ControllerBase
{
    private readonly IHubContext<ChatHub> _hubContext;

    public NotificationsController(IHubContext<ChatHub> hubContext)
    {
        _hubContext = hubContext;
    }

    [HttpPost("broadcast")]
    public async Task<IActionResult> Broadcast([FromBody] BroadcastRequest req)
    {
        // Sends to all clients across ALL server instances via Redis
        await _hubContext.Clients.All.SendAsync("SystemNotification", req.Message);
        return Ok();
    }

    [HttpPost("group/{groupName}")]
    public async Task<IActionResult> NotifyGroup(
        string groupName,
        [FromBody] BroadcastRequest req)
    {
        await _hubContext.Clients.Group(groupName).SendAsync("GroupMessage", req.Message);
        return Ok();
    }
}

public record BroadcastRequest(string Message);
```

## JavaScript Client

```javascript
import * as signalR from "@microsoft/signalr";

const connection = new signalR.HubConnectionBuilder()
  .withUrl("/chatHub")
  .withAutomaticReconnect()
  .build();

connection.on("ReceiveMessage", (payload) => {
  console.log(`${payload.user}: ${payload.message}`);
});

connection.on("SystemNotification", (message) => {
  showNotification(message);
});

await connection.start();
await connection.invoke("JoinRoom", "general");
await connection.invoke("SendMessage", "general", "Alice", "Hello everyone!");
```

## Redis Backplane with Authentication

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis(options =>
    {
        options.Configuration.EndPoints.Add("localhost", 6379);
        options.Configuration.Password = builder.Configuration["Redis:Password"];
        options.Configuration.Ssl = true;
        options.Configuration.AbortOnConnectFail = false;
        options.Configuration.ConnectTimeout = 5000;
        options.Configuration.ReconnectRetryPolicy = new ExponentialRetry(5000);
    });
```

## Summary

The Redis backplane for ASP.NET Core SignalR is enabled by calling `AddStackExchangeRedis()` after `AddSignalR()`. Once configured, all hub method calls that target groups, all users, or specific users are automatically published to Redis and consumed by all server instances, ensuring that messages reach every connected client regardless of which server they are connected to. This pattern is essential for any horizontally scaled real-time application.
