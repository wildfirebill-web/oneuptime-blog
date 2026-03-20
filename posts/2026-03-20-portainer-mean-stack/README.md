# How to Deploy a MEAN Stack via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, MEAN Stack, MongoDB, Express, Angular, Node.js, Docker Compose

Description: Deploy a complete MEAN (MongoDB, Express, Angular, Node.js) stack using Docker Compose through Portainer, with hot-reload for development and production-ready configuration.

## Introduction

The MEAN stack (MongoDB, Express.js, Angular, Node.js) is a popular JavaScript-centric full-stack framework where all components share a single language. Deploying it through Portainer using Docker Compose ensures consistent environments across development and production. This guide covers both a development configuration (with hot-reload) and a production setup.

## Prerequisites

- Portainer CE or BE installed and running
- Docker Engine 20.10+
- Familiarity with Node.js and Angular basics

## Step 1: Project Structure

Before creating the stack, organize your project:

```text
mean-app/
├── frontend/          # Angular application
│   ├── Dockerfile
│   └── ...
├── backend/           # Express + Node.js API
│   ├── Dockerfile
│   └── ...
└── docker-compose.yml
```

## Step 2: Create the Docker Compose Stack in Portainer

Navigate to **Stacks** → **Add Stack** → **Web Editor** and name it `mean-app`:

```yaml
version: "3.8"

services:
  # MongoDB database
  mongodb:
    image: mongo:7.0
    container_name: mean-mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: adminpassword
      MONGO_INITDB_DATABASE: meanapp
    volumes:
      - mongo_data:/data/db
      - ./mongo/init.js:/docker-entrypoint-initdb.d/init.js:ro
    networks:
      - mean-net

  # Node.js + Express backend API
  backend:
    image: node:20-alpine
    container_name: mean-backend
    restart: unless-stopped
    working_dir: /app
    volumes:
      - ./backend:/app
      - /app/node_modules       # Prevent host node_modules override
    command: sh -c "npm install && npm run dev"
    environment:
      NODE_ENV: development
      PORT: 3000
      MONGODB_URI: mongodb://admin:adminpassword@mongodb:27017/meanapp?authSource=admin
      JWT_SECRET: your-jwt-secret-change-in-production
    ports:
      - "3000:3000"
    depends_on:
      - mongodb
    networks:
      - mean-net

  # Angular frontend
  frontend:
    image: node:20-alpine
    container_name: mean-frontend
    restart: unless-stopped
    working_dir: /app
    volumes:
      - ./frontend:/app
      - /app/node_modules
    command: sh -c "npm install && npx ng serve --host 0.0.0.0 --port 4200"
    environment:
      NODE_ENV: development
    ports:
      - "4200:4200"
    depends_on:
      - backend
    networks:
      - mean-net

  # Mongo Express - web-based MongoDB admin UI
  mongo-express:
    image: mongo-express:latest
    container_name: mean-mongo-express
    restart: unless-stopped
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin
      ME_CONFIG_MONGODB_ADMINPASSWORD: adminpassword
      ME_CONFIG_MONGODB_URL: mongodb://admin:adminpassword@mongodb:27017/
      ME_CONFIG_BASICAUTH: "false"
    depends_on:
      - mongodb
    networks:
      - mean-net

volumes:
  mongo_data:
    driver: local

networks:
  mean-net:
    driver: bridge
```

## Step 3: MongoDB Initialization Script

```javascript
// mongo/init.js - creates the application database and user
db = db.getSiblingDB('meanapp');

db.createUser({
  user: 'meanuser',
  pwd: 'meanpassword',
  roles: [{ role: 'readWrite', db: 'meanapp' }]
});

// Create initial collections
db.createCollection('users');
db.createCollection('items');

print('Database initialized');
```

## Step 4: Express Backend Application

```javascript
// backend/src/app.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(cors({ origin: 'http://localhost:4200' }));
app.use(express.json());

// MongoDB connection
mongoose.connect(process.env.MONGODB_URI)
  .then(() => console.log('Connected to MongoDB'))
  .catch(err => console.error('MongoDB connection error:', err));

// Health check endpoint
app.get('/api/health', (req, res) => {
  res.json({
    status: 'ok',
    mongodb: mongoose.connection.readyState === 1 ? 'connected' : 'disconnected'
  });
});

// Sample items API
const Item = mongoose.model('Item', new mongoose.Schema({
  name: String,
  createdAt: { type: Date, default: Date.now }
}));

app.get('/api/items', async (req, res) => {
  const items = await Item.find();
  res.json(items);
});

app.post('/api/items', async (req, res) => {
  const item = new Item(req.body);
  await item.save();
  res.status(201).json(item);
});

app.listen(PORT, () => console.log(`Backend running on port ${PORT}`));
```

## Step 5: Angular Frontend Environment Configuration

```typescript
// frontend/src/environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:3000/api'
};
```

```typescript
// frontend/src/app/services/items.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { environment } from '../../environments/environment';

@Injectable({ providedIn: 'root' })
export class ItemsService {
  constructor(private http: HttpClient) {}

  getItems() {
    return this.http.get(`${environment.apiUrl}/items`);
  }
}
```

## Step 6: Production Docker Images

For production, build and use actual Docker images instead of running npm install at startup:

```dockerfile
# backend/Dockerfile

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app .
EXPOSE 3000
CMD ["node", "src/app.js"]
```

```dockerfile
# frontend/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx ng build --configuration=production

FROM nginx:alpine
COPY --from=builder /app/dist/frontend/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

## Step 7: Verify the Stack

```bash
# Verify all containers are healthy
docker ps | grep mean

# Test the backend API
curl http://localhost:3000/api/health

# Test MongoDB connectivity through the API
curl http://localhost:3000/api/items

# Check logs from Portainer:
# Stacks → mean-app → click each service to view logs

# Direct MongoDB access
docker exec -it mean-mongodb mongosh -u admin -p adminpassword
```

## Conclusion

Deploying the MEAN stack via Portainer provides a consistent, containerized environment with MongoDB data persisted in named volumes. The development configuration enables hot-reload for both the Angular frontend and Express backend. For production, swap the development service configurations for pre-built Docker images with multi-stage builds to reduce image size. Mongo Express provides a convenient browser-based interface for database administration, and all credentials should be moved to Portainer's environment variable store before going to production.
