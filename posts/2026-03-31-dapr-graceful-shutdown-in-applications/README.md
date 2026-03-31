# How to Implement Graceful Shutdown in Dapr Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Graceful Shutdown, Kubernetes, Reliability, Lifecycle

Description: Learn how to implement graceful shutdown in Dapr applications to drain in-flight requests, complete active subscriptions, and release resources cleanly before termination.

---

## Why Graceful Shutdown Matters

When Kubernetes terminates a pod, it sends SIGTERM and then waits for a configurable grace period before sending SIGKILL. Without graceful shutdown, your Dapr application may drop in-flight HTTP requests, lose pub/sub messages mid-processing, or leave state stores in an inconsistent state. Implementing graceful shutdown ensures clean termination.

## Dapr's Built-In Graceful Shutdown

Dapr sidecar already handles part of this: it stops accepting new traffic and drains existing connections when it receives SIGTERM. You control the sidecar shutdown timeout with the `dapr.io/graceful-shutdown-seconds` annotation:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/graceful-shutdown-seconds: "30"
```

## Go: Handling SIGTERM in a Dapr Service

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"

    daprd "github.com/dapr/go-sdk/service/http"
)

type Server struct {
    service daprd.Service
    wg      sync.WaitGroup
    mu      sync.Mutex
    active  int
}

func (s *Server) healthHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    fmt.Fprint(w, "OK")
}

func main() {
    s := daprd.NewService(":6001")

    // Register pub/sub subscription handler with graceful tracking
    s.AddTopicEventHandler(&common.Subscription{
        PubsubName: "messagebus",
        Topic:      "orders",
    }, func(ctx context.Context, e *common.TopicEvent) (retry bool, err error) {
        // Signal that we're processing a message
        wg := &sync.WaitGroup{}
        wg.Add(1)
        defer wg.Done()

        fmt.Printf("Processing order event: %s\n", e.ID)
        if err := processOrder(ctx, e); err != nil {
            return true, err // Retry on error
        }
        return false, nil
    })

    // Set up signal handling for graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    // Start the service in background
    go func() {
        if err := s.Start(); err != nil {
            log.Printf("Service error: %v", err)
        }
    }()

    // Wait for termination signal
    sig := <-quit
    fmt.Printf("Received signal %s, initiating graceful shutdown\n", sig)

    // Allow up to 25 seconds for in-flight work to finish
    ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
    defer cancel()

    if err := s.Stop(); err != nil {
        log.Printf("Error stopping service: %v", err)
    }

    fmt.Println("Service stopped cleanly")
}
```

## Python: Graceful Shutdown with FastAPI and Dapr

```python
import asyncio
import signal
import sys
from contextlib import asynccontextmanager
from fastapi import FastAPI
from dapr.ext.fastapi import DaprApp

# Track in-flight message processing
in_flight_tasks = set()
shutdown_event = asyncio.Event()

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    print("Service starting up")
    yield
    # Shutdown - wait for in-flight tasks
    print("Shutting down - waiting for in-flight tasks...")
    shutdown_event.set()
    if in_flight_tasks:
        await asyncio.wait(in_flight_tasks, timeout=25.0)
    print("All tasks completed, shutdown complete")

app = FastAPI(lifespan=lifespan)
dapr_app = DaprApp(app)

@dapr_app.subscribe(pubsub="messagebus", topic="orders")
async def handle_order(event_data: dict):
    task = asyncio.current_task()
    in_flight_tasks.add(task)
    try:
        print(f"Processing order: {event_data.get('id')}")
        await process_order(event_data)
        return {"success": True}
    finally:
        in_flight_tasks.discard(task)

def handle_sigterm(*args):
    print("SIGTERM received")
    loop = asyncio.get_event_loop()
    loop.create_task(graceful_shutdown())

signal.signal(signal.SIGTERM, handle_sigterm)
```

## .NET: Graceful Shutdown with IHostedService

```csharp
using Microsoft.Extensions.Hosting;
using System.Threading;

public class OrderProcessingService : BackgroundService
{
    private readonly ILogger<OrderProcessingService> _logger;
    private readonly SemaphoreSlim _activeWork = new SemaphoreSlim(0, 100);
    private int _activeCount = 0;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order processing service started");

        await Task.Delay(Timeout.Infinite, stoppingToken).ConfigureAwait(false);
    }

    public async Task ProcessOrderAsync(string orderId)
    {
        Interlocked.Increment(ref _activeCount);
        try
        {
            _logger.LogInformation("Processing order {OrderId}", orderId);
            await DoActualProcessing(orderId);
        }
        finally
        {
            int remaining = Interlocked.Decrement(ref _activeCount);
            if (remaining == 0) _activeWork.Release();
        }
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Graceful shutdown initiated, {Count} tasks in flight", _activeCount);

        // Wait up to 25 seconds for in-flight tasks
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
        cts.CancelAfter(TimeSpan.FromSeconds(25));

        while (_activeCount > 0 && !cts.Token.IsCancellationRequested)
        {
            await Task.Delay(500, cts.Token).ConfigureAwait(false);
        }

        _logger.LogInformation("Graceful shutdown complete");
        await base.StopAsync(cancellationToken);
    }
}
```

## Kubernetes Configuration

Align the Kubernetes termination grace period with your application shutdown timeout:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/graceful-shutdown-seconds: "30"
    spec:
      terminationGracePeriodSeconds: 45  # Longer than Dapr timeout
      containers:
      - name: order-service
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]  # Allow load balancer to drain
```

## Summary

Graceful shutdown in Dapr applications requires coordination between the application, the Dapr sidecar, and Kubernetes. Configure `dapr.io/graceful-shutdown-seconds` on your deployment, set `terminationGracePeriodSeconds` higher than your shutdown timeout, and implement signal handlers in your application that wait for in-flight operations to complete before exiting. This combination ensures that rolling deployments and pod terminations do not drop messages or corrupt state.
