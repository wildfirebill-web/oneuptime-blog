# How to Build FastAPI WebSocket Chat with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, FastAPI, WebSocket, Pub/Sub, Python

Description: Build a real-time multi-room WebSocket chat application in FastAPI using Redis Pub/Sub to broadcast messages across multiple server instances.

---

## Introduction

A naive WebSocket chat implementation stores connections in memory, which breaks when you scale to multiple server processes. Redis Pub/Sub solves this: each server subscribes to Redis channels, and any message published to a channel is broadcast to all subscribers regardless of which server the recipient is connected to. This guide builds a scalable chat application using FastAPI and Redis.

## Installation

```bash
pip install fastapi uvicorn redis[asyncio] websockets
```

## Connection Manager

```python
# connection_manager.py
from typing import Dict, Set
from fastapi import WebSocket

class ConnectionManager:
    def __init__(self):
        # room -> set of websockets
        self.rooms: Dict[str, Set[WebSocket]] = {}

    async def connect(self, websocket: WebSocket, room: str):
        await websocket.accept()
        if room not in self.rooms:
            self.rooms[room] = set()
        self.rooms[room].add(websocket)

    def disconnect(self, websocket: WebSocket, room: str):
        if room in self.rooms:
            self.rooms[room].discard(websocket)

    async def broadcast_to_room(self, message: str, room: str):
        if room in self.rooms:
            dead = set()
            for ws in self.rooms[room]:
                try:
                    await ws.send_text(message)
                except Exception:
                    dead.add(ws)
            self.rooms[room] -= dead
```

## Redis Pub/Sub Manager

```python
# redis_pubsub.py
import asyncio
import json
import redis.asyncio as aioredis

class RedisPubSub:
    def __init__(self, redis_url: str):
        self.redis_url = redis_url
        self.redis: aioredis.Redis = None

    async def connect(self):
        self.redis = aioredis.from_url(self.redis_url, decode_responses=True)

    async def publish(self, room: str, message: dict):
        await self.redis.publish(f"chat:{room}", json.dumps(message))

    async def subscribe(self, room: str, callback):
        pubsub = self.redis.pubsub()
        await pubsub.subscribe(f"chat:{room}")
        async for msg in pubsub.listen():
            if msg["type"] == "message":
                await callback(msg["data"])
```

## FastAPI Application

```python
# main.py
import asyncio
import json
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from connection_manager import ConnectionManager
from redis_pubsub import RedisPubSub

app = FastAPI()
manager = ConnectionManager()
pubsub = RedisPubSub("redis://localhost:6379")

@app.on_event("startup")
async def startup():
    await pubsub.connect()

@app.websocket("/ws/{room}/{username}")
async def websocket_endpoint(websocket: WebSocket, room: str, username: str):
    await manager.connect(websocket, room)

    # Subscribe to Redis channel and forward to local connections
    async def on_redis_message(data: str):
        await manager.broadcast_to_room(data, room)

    # Start Redis subscription in background
    subscribe_task = asyncio.create_task(
        pubsub.subscribe(room, on_redis_message)
    )

    try:
        while True:
            data = await websocket.receive_text()
            message = {
                "username": username,
                "room": room,
                "message": data,
            }
            # Publish to Redis - all server instances will receive this
            await pubsub.publish(room, message)
    except WebSocketDisconnect:
        manager.disconnect(websocket, room)
        subscribe_task.cancel()
        await manager.broadcast_to_room(
            json.dumps({"system": f"{username} left the room"}),
            room
        )
```

## HTML Client for Testing

```python
@app.get("/")
async def index():
    return {"message": "WebSocket chat server running"}

# Simple test client - serve as static HTML
CHAT_HTML = """
<!DOCTYPE html>
<html>
<body>
<input id="room" placeholder="Room name" value="general">
<input id="username" placeholder="Username" value="user1">
<button onclick="connect()">Connect</button>
<div id="messages"></div>
<input id="msg" placeholder="Message">
<button onclick="sendMsg()">Send</button>
<script>
let ws;
function connect() {
  const room = document.getElementById("room").value;
  const user = document.getElementById("username").value;
  ws = new WebSocket(`ws://localhost:8000/ws/${room}/${user}`);
  ws.onmessage = (e) => {
    const d = document.getElementById("messages");
    d.innerHTML += "<p>" + e.data + "</p>";
  };
}
function sendMsg() {
  ws.send(document.getElementById("msg").value);
}
</script>
</body>
</html>
"""
```

## Running the Server

```bash
uvicorn main:app --reload --workers 4
```

## Summary

Building a scalable FastAPI WebSocket chat with Redis Pub/Sub involves a local `ConnectionManager` for in-process WebSocket management and a `RedisPubSub` class to relay messages across server instances. Each server subscribes to a Redis channel per room, so messages published by any server instance reach all connected clients. This pattern scales horizontally - add more server instances behind a load balancer and Redis ensures message delivery across all of them.
