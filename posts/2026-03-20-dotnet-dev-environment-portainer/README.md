# How to Set Up a .NET Development Environment with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, .NET, C#, Development Environments, Docker, Hot Reload, Debugging

Description: Learn how to set up a .NET development environment with hot-reload and VS Code debugging in a Docker container managed by Portainer.

---

Running .NET development in Docker via Portainer ensures consistent SDK versions and eliminates environment mismatches. .NET's `dotnet watch` provides hot-reload for a native development experience.

## Dev Environment Compose Stack

```yaml
version: "3.8"

services:
  dotnet-dev:
    image: mcr.microsoft.com/dotnet/sdk:8.0-alpine
    restart: unless-stopped
    ports:
      - "5000:5000"    # HTTP
      - "5001:5001"    # HTTPS
      - "4024:4024"    # vsdbg debugger
    environment:
      DOTNET_ENVIRONMENT: Development
      DOTNET_WATCH_SUPPRESS_BROWSER_REFRESH: "1"
      ASPNETCORE_URLS: http://+:5000
    volumes:
      - ./app:/app
      # Cache NuGet packages
      - nuget_cache:/root/.nuget
    working_dir: /app
    # Watch for changes and auto-reload
    command: dotnet watch run --project MyApp.csproj --no-launch-profile

volumes:
  nuget_cache:
```

## Minimal ASP.NET Core Application

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.MapGet("/health", () => new { Status = "ok", Environment = app.Environment.EnvironmentName });

// Edit and save - dotnet watch reloads automatically
app.MapGet("/", () => "Hello from .NET dev environment");

app.Run();
```

## VS Code Remote Debugging

Install vsdbg inside the container:

```bash
# Via Portainer Exec console:

curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l /vsdbg
```

```json
// .vscode/launch.json
{
  "configurations": [
    {
      "name": "Attach to Container",
      "type": "coreclr",
      "request": "attach",
      "processId": "${command:pickRemoteProcess}",
      "pipeTransport": {
        "pipeProgram": "docker",
        "pipeArgs": ["exec", "-i", "dotnet-dev"],
        "debuggerPath": "/vsdbg/vsdbg"
      },
      "sourceFileMap": {
        "/app": "${workspaceFolder}/app"
      }
    }
  ]
}
```

## .csproj Configuration

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```
