# How to Use Dapr Go SDK with Chi Router

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Go, Chi Router, Http, Microservice, Sdk

Description: Integrate the Dapr Go SDK with the Chi router to build Dapr-enabled HTTP services that support pub/sub subscriptions alongside custom routing middleware.

---

## Overview

While the Dapr Go SDK ships its own HTTP service wrapper, you may prefer to use a feature-rich router like Chi for middleware support, URL parameter parsing, and sub-routing. This guide shows how to register Dapr's required subscription endpoints on a Chi router alongside your own routes.

## Installation

```bash
go get github.com/go-chi/chi/v5@latest
go get github.com/dapr/go-sdk/client@latest
```

## Setting Up Chi with Dapr

Dapr requires two HTTP endpoints from your service:
- `GET /dapr/subscribe` - returns the list of topic subscriptions
- Routes that handle incoming topic events (CloudEvents)

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "net/http"

    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
    dapr "github.com/dapr/go-sdk/client"
)

type Subscription struct {
    PubsubName string `json:"pubsubname"`
    Topic      string `json:"topic"`
    Route      string `json:"route"`
}

var daprClient dapr.Client

func main() {
    var err error
    daprClient, err = dapr.NewClient()
    if err != nil {
        log.Fatal(err)
    }
    defer daprClient.Close()

    r := chi.NewRouter()
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)

    // Dapr subscription registration endpoint
    r.Get("/dapr/subscribe", subscribeHandler)

    // Business routes
    r.Get("/products/{id}", getProduct)
    r.Post("/orders", createOrder)

    // Dapr topic event handlers
    r.Post("/events/order-placed", handleOrderPlaced)

    log.Println("Starting on :8080")
    http.ListenAndServe(":8080", r)
}
```

## Subscription Registration Handler

```go
func subscribeHandler(w http.ResponseWriter, r *http.Request) {
    subs := []Subscription{
        {PubsubName: "pubsub", Topic: "order-placed", Route: "/events/order-placed"},
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(subs)
}
```

## Handling CloudEvents with Chi

Dapr sends events as CloudEvents. Decode the CloudEvent envelope manually:

```go
type CloudEvent[T any] struct {
    ID          string `json:"id"`
    Source      string `json:"source"`
    Type        string `json:"type"`
    DataContent string `json:"datacontenttype"`
    Data        T      `json:"data"`
}

type Order struct {
    ID      string  `json:"id"`
    Product string  `json:"product"`
    Amount  float64 `json:"amount"`
}

func handleOrderPlaced(w http.ResponseWriter, r *http.Request) {
    var envelope CloudEvent[Order]
    if err := json.NewDecoder(r.Body).Decode(&envelope); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    order := envelope.Data
    log.Printf("Order placed: %s - %s ($%.2f)", order.ID, order.Product, order.Amount)

    // Persist order
    data, _ := json.Marshal(order)
    if err := daprClient.SaveState(r.Context(), "statestore", "order:"+order.ID, data, nil); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.WriteHeader(http.StatusOK)
}
```

## Chi Middleware for Request ID Propagation

```go
func daprTraceMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        traceID := r.Header.Get("traceparent")
        ctx := context.WithValue(r.Context(), "traceID", traceID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## Summary

Integrating Dapr with Chi requires manually implementing the `/dapr/subscribe` endpoint and handling CloudEvents JSON decoding in your route handlers. In exchange, you gain full Chi middleware support, structured sub-routers, and URL parameter helpers. The `DaprClient` is injected as a package-level variable or via a struct to keep handlers testable.
