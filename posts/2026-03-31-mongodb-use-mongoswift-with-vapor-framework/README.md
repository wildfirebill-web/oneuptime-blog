# How to Use MongoSwift with Vapor Framework

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Swift, Vapor, MongoSwift, REST

Description: Learn how to integrate MongoSwift into a Vapor application, register the MongoDB client as a service, and build type-safe REST routes.

---

## Introduction

Vapor is the most popular server-side Swift framework. The `MongoSwift` driver integrates naturally with Vapor through its `Application` storage system and `EventLoop`-compatible async model. Once configured, you can access MongoDB from any `Request` handler.

## Package Setup

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/vapor/vapor.git", from: "4.0.0"),
    .package(url: "https://github.com/mongodb/mongo-swift-driver", from: "1.3.0"),
],
targets: [
    .target(name: "App", dependencies: [
        .product(name: "Vapor",      package: "vapor"),
        .product(name: "MongoSwift", package: "mongo-swift-driver"),
    ])
]
```

## Registering the Client with Vapor

```swift
// Sources/App/configure.swift
import Vapor
import MongoSwift

extension Application {
    struct MongoClientKey: StorageKey {
        typealias Value = MongoClient
    }

    var mongoClient: MongoClient {
        get { storage[MongoClientKey.self]! }
        set { storage[MongoClientKey.self] = newValue }
    }
}

public func configure(_ app: Application) async throws {
    app.mongoClient = try MongoClient(
        Environment.get("MONGODB_URI") ?? "mongodb://localhost:27017",
        using: app.eventLoopGroup
    )

    try routes(app)

    app.lifecycle.use(MongoClientShutdown(app))
}

struct MongoClientShutdown: LifecycleHandler {
    let app: Application
    func shutdown(_ application: Application) {
        try? app.mongoClient.syncClose()
    }
}
```

## Model Definition

```swift
struct Product: Content {
    var id:       BSONObjectID?
    var name:     String
    var price:    Double
    var category: String

    enum CodingKeys: String, CodingKey {
        case id = "_id"
        case name, price, category
    }
}
```

## Route Handlers

```swift
// Sources/App/routes.swift
import Vapor
import MongoSwift

func routes(_ app: Application) throws {
    app.get("products") { req async throws -> [Product] in
        let col = req.application.mongoClient
            .db("shop").collection("products", withType: Product.self)
        var products: [Product] = []
        let cursor = try await col.find([:])
        for try await p in cursor { products.append(p) }
        return products
    }

    app.post("products") { req async throws -> HTTPStatus in
        let product = try req.content.decode(Product.self)
        let col = req.application.mongoClient
            .db("shop").collection("products", withType: Product.self)
        try await col.insertOne(product)
        return .created
    }

    app.get("products", ":id") { req async throws -> Product in
        guard let idStr = req.parameters.get("id"),
              let oid   = try? BSONObjectID(idStr) else {
            throw Abort(.badRequest)
        }
        let col = req.application.mongoClient
            .db("shop").collection("products", withType: Product.self)
        guard let product = try await col.findOne(["_id": .objectID(oid)]) else {
            throw Abort(.notFound)
        }
        return product
    }
}
```

## Summary

Integrating MongoSwift with Vapor involves registering a `MongoClient` in `Application.Storage`, implementing a `LifecycleHandler` for graceful shutdown, and accessing the client through `req.application.mongoClient` in route handlers. Vapor's `Content` protocol works directly with Codable structs, so MongoDB documents serialize cleanly to and from JSON responses.
