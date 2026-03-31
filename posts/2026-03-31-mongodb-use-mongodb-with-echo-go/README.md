# How to Use MongoDB with Echo (Go)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Go, Echo

Description: Integrate MongoDB with Echo, Go's high-performance web framework, using the official driver with middleware, dependency injection, and structured error handling.

---

## Echo and MongoDB

Echo is a fast, minimalist Go web framework known for its middleware ecosystem and clean router API. Combined with the official MongoDB Go driver, it provides a solid foundation for building REST APIs with structured error handling and dependency injection.

## Project Setup

```bash
mkdir echo-mongo && cd echo-mongo
go mod init echo-mongo
go get github.com/labstack/echo/v4
go get go.mongodb.org/mongo-driver/mongo
go get go.mongodb.org/mongo-driver/bson
```

## Application Structure

```text
echo-mongo/
  main.go
  db/
    mongo.go
  models/
    user.go
  handlers/
    user.go
```

## Database Package

`db/mongo.go`:

```go
package db

import (
    "context"
    "log"
    "os"
    "time"

    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

type MongoDB struct {
    Client *mongo.Client
    DB     *mongo.Database
}

func New() *MongoDB {
    uri := os.Getenv("MONGO_URI")
    if uri == "" {
        uri = "mongodb://localhost:27017"
    }

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    client, err := mongo.Connect(ctx, options.Client().ApplyURI(uri).SetMaxPoolSize(20))
    if err != nil {
        log.Fatal(err)
    }

    return &MongoDB{
        Client: client,
        DB:     client.Database(os.Getenv("MONGO_DB")),
    }
}

func (m *MongoDB) Collection(name string) *mongo.Collection {
    return m.DB.Collection(name)
}
```

## Model

`models/user.go`:

```go
package models

import (
    "time"
    "go.mongodb.org/mongo-driver/bson/primitive"
)

type User struct {
    ID        primitive.ObjectID `bson:"_id,omitempty" json:"id"`
    Name      string             `bson:"name" json:"name" validate:"required,min=2"`
    Email     string             `bson:"email" json:"email" validate:"required,email"`
    CreatedAt time.Time          `bson:"created_at" json:"createdAt"`
}
```

## Handler with Dependency Injection

`handlers/user.go`:

```go
package handlers

import (
    "context"
    "net/http"
    "time"

    "echo-mongo/db"
    "echo-mongo/models"
    "github.com/labstack/echo/v4"
    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
)

type UserHandler struct {
    db *db.MongoDB
}

func NewUserHandler(database *db.MongoDB) *UserHandler {
    return &UserHandler{db: database}
}

func (h *UserHandler) List(c echo.Context) error {
    ctx, cancel := context.WithTimeout(c.Request().Context(), 5*time.Second)
    defer cancel()

    cursor, err := h.db.Collection("users").Find(ctx, bson.M{})
    if err != nil {
        return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
    }
    defer cursor.Close(ctx)

    var users []models.User
    if err = cursor.All(ctx, &users); err != nil {
        return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
    }
    return c.JSON(http.StatusOK, users)
}

func (h *UserHandler) Create(c echo.Context) error {
    var u models.User
    if err := c.Bind(&u); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }
    u.ID = primitive.NewObjectID()
    u.CreatedAt = time.Now()

    ctx, cancel := context.WithTimeout(c.Request().Context(), 5*time.Second)
    defer cancel()

    _, err := h.db.Collection("users").InsertOne(ctx, u)
    if err != nil {
        if mongo.IsDuplicateKeyError(err) {
            return echo.NewHTTPError(http.StatusConflict, "email already exists")
        }
        return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
    }
    return c.JSON(http.StatusCreated, u)
}
```

## Main Application

```go
package main

import (
    "echo-mongo/db"
    "echo-mongo/handlers"
    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

func main() {
    database := db.New()

    e := echo.New()
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())

    uh := handlers.NewUserHandler(database)
    e.GET("/users", uh.List)
    e.POST("/users", uh.Create)

    e.Logger.Fatal(e.Start(":3000"))
}
```

## Summary

Echo with MongoDB uses dependency injection to pass the database client into handlers, keeping tests easy and concerns separated. Use context with timeout on every MongoDB call, handle duplicate key errors explicitly with `mongo.IsDuplicateKeyError`, and leverage Echo's `echo.NewHTTPError` for consistent error responses. This pattern scales well as you add more collections and handlers.
