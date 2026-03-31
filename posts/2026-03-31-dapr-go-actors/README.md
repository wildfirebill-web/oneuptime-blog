# How to Use Dapr Actors with Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Go, Actor, Microservice, Distributed System, Concurrency

Description: Implement the virtual actor pattern in Go using the Dapr actor building block for stateful, single-threaded distributed objects with timers and reminders.

---

## Overview

Dapr's actor building block implements the virtual actor pattern: actors are single-threaded objects with isolated state that are automatically activated on first call and garbage-collected when idle. The Dapr Go SDK provides an interface for defining actor types and registering them with the runtime.

## Defining an Actor Interface

```go
package actors

import "context"

type CounterActor interface {
    Increment(ctx context.Context) error
    GetCount(ctx context.Context) (int, error)
    Reset(ctx context.Context) error
}
```

## Implementing the Actor

```go
package actors

import (
    "context"
    "encoding/json"

    dapr "github.com/dapr/go-sdk/actor"
)

type CounterActorImpl struct {
    dapr.ServerImplBase
}

func (a *CounterActorImpl) Type() string { return "CounterActor" }

func (a *CounterActorImpl) Increment(ctx context.Context) error {
    count, err := a.getCount(ctx)
    if err != nil {
        return err
    }
    data, _ := json.Marshal(count + 1)
    return a.GetStateManager().Set(ctx, "count", data)
}

func (a *CounterActorImpl) GetCount(ctx context.Context) (int, error) {
    return a.getCount(ctx)
}

func (a *CounterActorImpl) Reset(ctx context.Context) error {
    data, _ := json.Marshal(0)
    return a.GetStateManager().Set(ctx, "count", data)
}

func (a *CounterActorImpl) getCount(ctx context.Context) (int, error) {
    raw, err := a.GetStateManager().Get(ctx, "count")
    if err != nil || raw == nil {
        return 0, nil
    }
    var count int
    json.Unmarshal(raw, &count)
    return count, nil
}
```

## Registering the Actor with the Service

```go
package main

import (
    "log"

    "github.com/dapr/go-sdk/actor/config"
    daprd "github.com/dapr/go-sdk/service/http"
    "myapp/actors"
)

func main() {
    s := daprd.NewService(":8080")

    s.RegisterActor(&actors.CounterActorImpl{})

    if err := s.Start(); err != nil {
        log.Fatal(err)
    }
}
```

## Calling an Actor from a Client

```go
proxy := client.NewActorProxy(
    dapr.NewActorID("counter-1"),
    "CounterActor",
)

// Increment
if err := proxy.Call(ctx, "Increment", nil); err != nil {
    log.Fatal(err)
}

// Get count
var count int
if err := proxy.CallWithResult(ctx, "GetCount", nil, &count); err != nil {
    log.Fatal(err)
}
log.Printf("Count: %d", count)
```

## Adding a Reminder

```go
func (a *CounterActorImpl) OnActivate() error {
    return a.RegisterActorReminder(&dapr.ActorReminder{
        Name:    "daily-reset",
        DueTime: "24h",
        Period:  "24h",
        Data:    nil,
    })
}

func (a *CounterActorImpl) ReminderCall(reminderName string, state []byte,
    dueTime string, period string) {
    if reminderName == "daily-reset" {
        a.Reset(context.Background())
    }
}
```

## Summary

The Dapr Go actor implementation requires three steps: defining an interface, implementing `ServerImplBase`, and registering with the service. The Dapr runtime handles placement, state persistence, and turn-based concurrency automatically. Reminders and timers make it easy to build self-managing stateful actors without external schedulers.
