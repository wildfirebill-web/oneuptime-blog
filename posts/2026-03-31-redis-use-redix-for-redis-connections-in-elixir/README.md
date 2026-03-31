# How to Use Redix for Redis Connections in Elixir

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Elixir, Redix, Connection Pooling, OTP

Description: Connect to Redis from Elixir using Redix, manage connection pools with Redix.ConnectionPool, and run commands in a supervised OTP application.

---

## Introduction

`Redix` is a low-level, fast Redis client for Elixir. It provides direct Redis protocol support and integrates naturally with OTP supervision trees. For production use, `Redix.ConnectionPool` manages a pool of connections so concurrent processes do not contend for a single connection. This guide covers single connections, pooling, and pipeline commands.

## Adding Dependencies

In `mix.exs`:

```elixir
defp deps do
  [
    {:redix, "~> 1.3"},
    {:castore, ">= 0.0.0"},  # For TLS support
  ]
end
```

```bash
mix deps.get
```

## Single Connection

Start a connection manually for development or testing:

```elixir
{:ok, conn} = Redix.start_link("redis://localhost:6379")

# SET a key
{:ok, "OK"} = Redix.command(conn, ["SET", "name", "Alice"])

# GET a key
{:ok, "Alice"} = Redix.command(conn, ["GET", "name"])

# DEL a key
{:ok, 1} = Redix.command(conn, ["DEL", "name"])
```

## Starting Redix Under a Supervisor

Add Redix to your application's supervision tree:

```elixir
# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      {Redix, {
        "redis://localhost:6379",
        [name: :redix, sync_connect: false]
      }},
      # ...other children
    ]

    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

Now call Redis from anywhere using the registered name:

```elixir
Redix.command(:redix, ["SET", "greeting", "hello"])
Redix.command(:redix, ["GET", "greeting"])
```

## Connection Pooling

Use a pool of connections for concurrent workloads:

```elixir
children = [
  {Redix.ConnectionPool, [
    url: "redis://localhost:6379",
    name: :redix_pool,
    pool_size: 10,
  ]},
]
```

```elixir
# Use the pool from any process
Redix.ConnectionPool.command(:redix_pool, ["SET", "key", "value"])
{:ok, val} = Redix.ConnectionPool.command(:redix_pool, ["GET", "key"])
```

## Pipelining Commands

Send multiple commands in a single network round-trip:

```elixir
commands = [
  ["SET", "counter", "0"],
  ["INCR", "counter"],
  ["INCR", "counter"],
  ["GET", "counter"],
]

{:ok, results} = Redix.pipeline(:redix, commands)
# results: ["OK", 1, 2, "2"]
```

## Working with Data Types

```elixir
# Hash operations
{:ok, _} = Redix.command(:redix, ["HSET", "user:1", "name", "Alice", "email", "alice@example.com"])
{:ok, name} = Redix.command(:redix, ["HGET", "user:1", "name"])
{:ok, fields} = Redix.command(:redix, ["HGETALL", "user:1"])

# List operations
{:ok, _} = Redix.command(:redix, ["RPUSH", "queue", "job1", "job2", "job3"])
{:ok, job} = Redix.command(:redix, ["LPOP", "queue"])

# Set with TTL
{:ok, _} = Redix.command(:redix, ["SET", "session:abc", "user_data", "EX", "3600"])
{:ok, ttl} = Redix.command(:redix, ["TTL", "session:abc"])
```

## Error Handling

```elixir
defmodule MyApp.Cache do
  def get(key) do
    case Redix.command(:redix, ["GET", key]) do
      {:ok, nil} ->
        {:miss, nil}
      {:ok, value} ->
        {:hit, value}
      {:error, %Redix.ConnectionError{reason: :closed}} ->
        {:error, :redis_unavailable}
      {:error, reason} ->
        {:error, reason}
    end
  end

  def set(key, value, ttl_seconds \\ 300) do
    Redix.command(:redix, ["SET", key, value, "EX", Integer.to_string(ttl_seconds)])
  end
end
```

## Connecting with Authentication and TLS

```elixir
{Redix, [
  host: "my-redis.example.com",
  port: 6380,
  password: System.get_env("REDIS_PASSWORD"),
  ssl: true,
  name: :redix_secure,
]}
```

## Summary

Redix is a lightweight, protocol-compliant Redis client for Elixir that integrates with OTP supervision trees. Starting Redix as a named child process allows any module in the application to issue commands via `Redix.command/3`. For concurrent applications, `Redix.ConnectionPool` manages multiple connections transparently. Pipeline commands with `Redix.pipeline/3` reduce round-trip latency when multiple operations must execute in sequence.
