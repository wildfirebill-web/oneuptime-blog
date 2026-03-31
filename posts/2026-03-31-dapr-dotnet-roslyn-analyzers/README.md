# How to Use Dapr .NET SDK Roslyn Analyzers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Dotnet, Roslyn, Analyzer, Code Quality, Sdk

Description: Use Dapr's built-in Roslyn analyzers to catch common misconfigurations and API misuse in your .NET projects at compile time before they reach production.

---

## Overview

The Dapr .NET SDK ships with a set of Roslyn analyzers that run during compilation and flag common mistakes such as incorrect actor interface definitions, missing base class implementations, and invalid method signatures. These analyzers provide immediate feedback in your IDE and CI pipeline without any runtime overhead.

## Enabling the Analyzers

The analyzers are included automatically when you install the Dapr Actor package:

```bash
dotnet add package Dapr.Actors
```

No additional configuration is required. The analyzers activate as soon as the package is referenced.

## Actor Interface Violations

Dapr actors must implement `IActor` and follow naming conventions. The analyzer catches violations at compile time:

```csharp
// This will trigger DAPR0001: Actor interface must extend IActor
public interface IMyActor
{
    Task DoWorkAsync();
}

// Correct version
public interface IMyActor : IActor
{
    Task DoWorkAsync();
}
```

## Actor Method Signature Rules

Actor methods must return `Task` or `Task<T>`, and must not have `out` or `ref` parameters. The analyzer enforces these rules:

```csharp
// DAPR0002: Actor methods must return Task or Task<T>
public interface ICounterActor : IActor
{
    int GetCount(); // Incorrect - must return Task<int>
    Task<int> GetCountAsync(); // Correct
}
```

## Viewing Analyzer Diagnostics

In Visual Studio or Rider, diagnostics appear as warnings or errors inline. In the CLI:

```bash
dotnet build --warnaserror:DAPR0001,DAPR0002
```

## Suppressing False Positives

If a rule does not apply to a specific case, suppress it with the standard `#pragma` directive:

```csharp
#pragma warning disable DAPR0001
public interface ILegacyActor
{
    Task OldMethodAsync();
}
#pragma warning restore DAPR0001
```

Or add a project-level suppression in your `.csproj`:

```xml
<PropertyGroup>
  <NoWarn>DAPR0001</NoWarn>
</PropertyGroup>
```

## Integrating with CI

Treat Dapr analyzer warnings as errors in your CI pipeline to prevent regressions:

```bash
dotnet build -p:TreatWarningsAsErrors=true
```

## Summary

Dapr's Roslyn analyzers give .NET teams compile-time safety for actor interface contracts and SDK usage patterns. By including these checks in CI and configuring `TreatWarningsAsErrors`, teams can prevent common Dapr misconfigurations from reaching production with zero runtime cost.
