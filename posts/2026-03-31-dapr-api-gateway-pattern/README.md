# How to Implement API Gateway Pattern with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, API Gateway, Pattern, Microservice, Service Invocation

Description: Learn how to implement the API gateway pattern using Dapr service invocation to build a centralized entry point for routing, auth, and aggregation.

---

## Overview

The API Gateway pattern provides a single entry point for clients to interact with multiple backend microservices. It handles cross-cutting concerns like authentication, rate limiting, request routing, and response aggregation. Dapr's service invocation building block makes it easy to build a robust gateway.

## Gateway Service Structure

The gateway receives external requests and fans out to internal Dapr-enabled microservices:

```go
package main

import (
    "context"
    "encoding/json"
    "net/http"
    dapr "github.com/dapr/go-sdk/client"
    "github.com/gorilla/mux"
)

type Gateway struct {
    daprClient dapr.Client
}

func main() {
    client, _ := dapr.NewClient()
    defer client.Close()

    gw := &Gateway{daprClient: client}
    r := mux.NewRouter()

    // Route to backend microservices
    r.HandleFunc("/api/users/{id}", gw.authenticate(gw.getUser)).Methods("GET")
    r.HandleFunc("/api/orders", gw.authenticate(gw.createOrder)).Methods("POST")
    r.HandleFunc("/api/dashboard", gw.authenticate(gw.getDashboard)).Methods("GET")

    http.ListenAndServe(":8080", r)
}
```

## Authentication Middleware

```go
func (gw *Gateway) authenticate(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }

        // Invoke auth service via Dapr
        result, err := gw.daprClient.InvokeMethodWithContent(
            context.Background(),
            "auth-service",
            "/validate",
            "POST",
            &dapr.DataContent{
                ContentType: "application/json",
                Data:        []byte(`{"token":"` + token + `"}`),
            },
        )
        if err != nil || string(result) != `{"valid":true}` {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }

        next(w, r)
    }
}
```

## Routing to Backend Services

```go
func (gw *Gateway) getUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID := vars["id"]

    data, err := gw.daprClient.InvokeMethod(
        context.Background(),
        "user-service",
        "/users/"+userID,
        "GET",
    )
    if err != nil {
        http.Error(w, "service unavailable", 503)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(data)
}

func (gw *Gateway) createOrder(w http.ResponseWriter, r *http.Request) {
    var body map[string]interface{}
    json.NewDecoder(r.Body).Decode(&body)
    bodyBytes, _ := json.Marshal(body)

    result, err := gw.daprClient.InvokeMethodWithContent(
        context.Background(),
        "order-service",
        "/orders",
        "POST",
        &dapr.DataContent{ContentType: "application/json", Data: bodyBytes},
    )
    if err != nil {
        http.Error(w, "order service error", 500)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    w.Write(result)
}
```

## Response Aggregation (BFF)

The gateway can aggregate multiple service calls into a single response:

```go
func (gw *Gateway) getDashboard(w http.ResponseWriter, r *http.Request) {
    ctx := context.Background()

    // Call multiple services in parallel
    userCh := make(chan []byte, 1)
    orderCh := make(chan []byte, 1)
    statsCh := make(chan []byte, 1)

    go func() {
        data, _ := gw.daprClient.InvokeMethod(ctx, "user-service", "/me", "GET")
        userCh <- data
    }()
    go func() {
        data, _ := gw.daprClient.InvokeMethod(ctx, "order-service", "/recent", "GET")
        orderCh <- data
    }()
    go func() {
        data, _ := gw.daprClient.InvokeMethod(ctx, "stats-service", "/summary", "GET")
        statsCh <- data
    }()

    dashboard := map[string]json.RawMessage{
        "user":   <-userCh,
        "orders": <-orderCh,
        "stats":  <-statsCh,
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(dashboard)
}
```

## Rate Limiting with Dapr State

```go
func (gw *Gateway) rateLimit(clientIP string) bool {
    key := "ratelimit:" + clientIP
    item, _ := gw.daprClient.GetState(context.Background(), "statestore", key, nil)
    var count int
    if item.Value != nil {
        json.Unmarshal(item.Value, &count)
    }
    if count >= 100 {
        return false
    }
    count++
    countBytes, _ := json.Marshal(count)
    gw.daprClient.SaveState(context.Background(), "statestore", key, countBytes, nil)
    return true
}
```

## Summary

The API gateway pattern with Dapr centralizes cross-cutting concerns like authentication, routing, and aggregation. Dapr service invocation handles service discovery and mTLS between the gateway and backend services automatically. Response aggregation reduces client round trips while Dapr's state store enables stateful features like rate limiting.
