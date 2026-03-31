# How to Use MongoDB with Fiber (Go)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Go, Fiber

Description: Build a REST API with Go's Fiber framework connected to MongoDB using the official Go driver, with CRUD handlers, connection pooling, and error handling.

---

## Fiber and MongoDB

Fiber is an Express-inspired Go web framework built on Fasthttp. It is fast, low-overhead, and simple to use. Paired with the official MongoDB Go driver, it makes for a productive stack for building high-performance APIs.

## Project Setup

```bash
mkdir fiber-mongo && cd fiber-mongo
go mod init fiber-mongo
go get github.com/gofiber/fiber/v2
go get go.mongodb.org/mongo-driver/mongo
go get go.mongodb.org/mongo-driver/bson
```

## MongoDB Client

Create `database/mongo.go`:

```go
package database

import (
    "context"
    "log"
    "os"
    "time"

    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

var DB *mongo.Database

func Connect() {
    uri := os.Getenv("MONGO_URI")
    if uri == "" {
        uri = "mongodb://localhost:27017"
    }

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    opts := options.Client().ApplyURI(uri).
        SetMaxPoolSize(20).
        SetMinPoolSize(5).
        SetServerSelectionTimeout(5 * time.Second)

    client, err := mongo.Connect(ctx, opts)
    if err != nil {
        log.Fatal("MongoDB connect error:", err)
    }

    if err = client.Ping(ctx, nil); err != nil {
        log.Fatal("MongoDB ping error:", err)
    }

    DB = client.Database(os.Getenv("MONGO_DB"))
    log.Println("MongoDB connected")
}
```

## Model and Handler

Create `handlers/product.go`:

```go
package handlers

import (
    "context"
    "time"

    "fiber-mongo/database"
    "github.com/gofiber/fiber/v2"
    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo/options"
)

type Product struct {
    ID        primitive.ObjectID `bson:"_id,omitempty" json:"id"`
    Name      string             `bson:"name" json:"name"`
    Price     float64            `bson:"price" json:"price"`
    Category  string             `bson:"category" json:"category"`
    CreatedAt time.Time          `bson:"created_at" json:"createdAt"`
}

func GetProducts(c *fiber.Ctx) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    col := database.DB.Collection("products")
    cursor, err := col.Find(ctx, bson.M{}, options.Find().SetLimit(50))
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": err.Error()})
    }
    defer cursor.Close(ctx)

    var products []Product
    if err = cursor.All(ctx, &products); err != nil {
        return c.Status(500).JSON(fiber.Map{"error": err.Error()})
    }
    return c.JSON(products)
}

func CreateProduct(c *fiber.Ctx) error {
    var p Product
    if err := c.BodyParser(&p); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "invalid body"})
    }
    p.ID = primitive.NewObjectID()
    p.CreatedAt = time.Now()

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    col := database.DB.Collection("products")
    _, err := col.InsertOne(ctx, p)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": err.Error()})
    }
    return c.Status(201).JSON(p)
}
```

## Main Application

```go
package main

import (
    "fiber-mongo/database"
    "fiber-mongo/handlers"
    "github.com/gofiber/fiber/v2"
)

func main() {
    database.Connect()

    app := fiber.New()

    api := app.Group("/api")
    api.Get("/products", handlers.GetProducts)
    api.Post("/products", handlers.CreateProduct)

    app.Listen(":3000")
}
```

## Running the Application

```bash
MONGO_URI="mongodb://localhost:27017" MONGO_DB="fiberapp" go run main.go
```

## Summary

Fiber with the MongoDB Go driver provides a concise, high-performance API stack. Use a shared database package to manage connection pooling, define struct types with BSON tags for serialization, and use context with timeout on every MongoDB operation to prevent hanging requests. The Fiber routing model mirrors Express patterns, making it approachable while delivering Go's concurrency performance.
