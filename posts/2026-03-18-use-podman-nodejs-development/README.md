# How to Use Podman for Node.js Development

Author: [nawazdhandala](https://github.com/nawazdhandala)

Tags: Podman, Node.js, JavaScript, Containers, Development

Description: A hands-on guide to using Podman for Node.js development, covering project setup, dependency management, Express and Next.js workflows, testing, and debugging.

---

> Podman keeps your Node.js development environment isolated and reproducible, so you never have to worry about node version mismatches or global package conflicts again.

Node.js development has a well-known problem: version drift. One project needs Node 18, another needs Node 20, and a third depends on a specific npm version. Tools like nvm help, but they only manage the Node binary. System-level dependencies, native modules, and global packages still cause trouble. Running Node.js inside Podman containers eliminates all of these issues. Each project gets its own complete environment, and your host machine stays clean.

This guide walks through practical Podman workflows for Node.js, from running simple scripts to building full-stack applications with databases.

---

## Choosing a Node.js Base Image

The official Node.js images come in several variants:

```bash
# Full image - Debian-based, includes build tools for native modules

podman pull docker.io/library/node:20

# Slim image - smaller Debian-based, no build tools
podman pull docker.io/library/node:20-slim

# Alpine image - smallest, but native modules may need extra work
podman pull docker.io/library/node:20-alpine
```

For development, `node:20-slim` works well for most projects. If your project uses native modules (like `bcrypt`, `sharp`, or `sqlite3`), use the full `node:20` image or install build tools in the slim image.

## Running a Node.js Script

The quickest way to run Node.js code in Podman:

```bash
# Run a single script
podman run --rm \
  -v $(pwd):/app:Z \
  -w /app \
  docker.io/library/node:20-slim \
  node index.js

# Start an interactive Node.js REPL
podman run -it --rm \
  docker.io/library/node:20-slim \
  node
```

## Setting Up an Express.js Project

Here is a complete workflow for developing an Express application inside Podman.

Create `package.json`:

```json
{
  "name": "express-podman-demo",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.21.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.0",
    "jest": "^29.7.0"
  }
}
```

Create `server.js`:

```javascript
// server.js
const express = require("express");
const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get("/", (req, res) => {
  res.json({ message: "Hello from Express inside Podman" });
});

app.get("/health", (req, res) => {
  res.json({ status: "healthy", uptime: process.uptime() });
});

app.listen(PORT, "0.0.0.0", () => {
  console.log(`Server running on port ${PORT}`);
});
```

Create a `Containerfile`:

```dockerfile
FROM docker.io/library/node:20-slim

WORKDIR /app

# Copy package files first for layer caching
COPY package.json package-lock.json* ./

# Install dependencies
RUN npm ci

# Copy application code
COPY . .

EXPOSE 3000

# Use nodemon for development (auto-restart on file changes)
CMD ["npx", "nodemon", "server.js"]
```

Build and run:

```bash
# Build the development image
podman build -t express-dev .

# Run with source code mounted for live editing
podman run -it --rm \
  -v $(pwd):/app:Z \
  -v /app/node_modules \
  -p 3000:3000 \
  express-dev
```

The second `-v /app/node_modules` flag is important. It creates an anonymous volume for `node_modules`, preventing the host's `node_modules` directory (if it exists) from overriding the container's installed dependencies. This avoids platform-specific binary mismatches between your host OS and the Linux container.

## Handling node_modules Correctly

The `node_modules` conflict between host and container is the most common pain point in containerized Node.js development. Here are two approaches:

### Approach 1: Anonymous Volume (Recommended)

```bash
# Mount source code but keep container's node_modules
podman run -it --rm \
  -v $(pwd):/app:Z \
  -v /app/node_modules \
  -p 3000:3000 \
  express-dev
```

This mounts your source code into the container but uses the container's own `node_modules`. The downside is that your editor on the host cannot see the installed packages for autocomplete.

### Approach 2: Install in Container, Sync to Host

```bash
# Install dependencies inside the container and copy them back
podman run --rm \
  -v $(pwd):/app:Z \
  -w /app \
  docker.io/library/node:20-slim \
  npm install

# Now node_modules exists on the host too
```

This installs Linux-compatible binaries, which may not work if your host is macOS or Windows. It works well when the host and container share the same OS architecture.

## Developing a Next.js Application

Next.js works well with Podman. Create a `Containerfile` for Next.js development:

```dockerfile
FROM docker.io/library/node:20-slim

WORKDIR /app

COPY package.json package-lock.json* ./
RUN npm ci

COPY . .

EXPOSE 3000

# Next.js dev server with hot reload
CMD ["npx", "next", "dev"]
```

Run with the proper environment variables for hot module replacement:

```bash
# Build the image
podman build -t nextjs-dev .

# Run with HMR support
podman run -it --rm \
  -v $(pwd):/app:Z \
  -v /app/node_modules \
  -v /app/.next \
  -p 3000:3000 \
  -e WATCHPACK_POLLING=true \
  nextjs-dev
```

The `WATCHPACK_POLLING=true` environment variable enables polling-based file watching, which is necessary when the source code is mounted from the host because filesystem events do not always propagate across the mount boundary.

## Multi-Container Setup with a Database

Most Node.js applications need a database. Here is a `docker-compose.yml` for an Express app with MongoDB:

```yaml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - .:/app:Z
      - /app/node_modules
    environment:
      MONGO_URL: mongodb://mongo:27017/myapp
      NODE_ENV: development
    depends_on:
      - mongo

  mongo:
    image: docker.io/library/mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongodata:/data/db

volumes:
  mongodata:
```

Start the stack:

```bash
podman-compose up -d

# Check logs
podman-compose logs -f app

# Connect to MongoDB shell
podman-compose exec mongo mongosh
```

## Running Tests

Run your test suite inside a container for consistency:

```bash
# Run Jest tests
podman run --rm \
  -v $(pwd):/app:Z \
  -v /app/node_modules \
  -w /app \
  express-dev \
  npx jest --verbose

# Run tests with coverage
podman run --rm \
  -v $(pwd):/app:Z \
  -v /app/node_modules \
  -w /app \
  express-dev \
  npx jest --coverage
```

## Debugging Node.js in a Container

Node.js has a built-in inspector that you can expose from the container:

```bash
# Run with Node.js inspector enabled
podman run -it --rm \
  -v $(pwd):/app:Z \
  -v /app/node_modules \
  -p 3000:3000 \
  -p 9229:9229 \
  express-dev \
  node --inspect=0.0.0.0:9229 server.js
```

Then open `chrome://inspect` in Chrome or attach VS Code's debugger to `localhost:9229`. Your VS Code `launch.json` for attaching:

```json
{
  "type": "node",
  "request": "attach",
  "name": "Attach to Container",
  "port": 9229,
  "localRoot": "${workspaceFolder}",
  "remoteRoot": "/app"
}
```

## Testing Against Multiple Node.js Versions

Run your tests against different Node.js versions without installing any of them locally:

```bash
# Test against Node.js 18 LTS
podman run --rm -v $(pwd):/app:Z -w /app \
  docker.io/library/node:18-slim \
  bash -c "npm ci && npm test"

# Test against Node.js 20 LTS
podman run --rm -v $(pwd):/app:Z -w /app \
  docker.io/library/node:20-slim \
  bash -c "npm ci && npm test"

# Test against Node.js 22
podman run --rm -v $(pwd):/app:Z -w /app \
  docker.io/library/node:22-slim \
  bash -c "npm ci && npm test"
```

## Building a Production Image

Once development is done, build a lean production image:

```dockerfile
# Multi-stage build for production
FROM docker.io/library/node:20-slim AS builder
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
COPY . .
RUN npm run build

FROM docker.io/library/node:20-slim
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci --omit=dev
COPY --from=builder /app/dist ./dist
EXPOSE 3000
USER node
CMD ["node", "dist/server.js"]
```

```bash
# Build the production image
podman build -t express-prod -f Containerfile.prod .

# Test it locally
podman run --rm -p 3000:3000 express-prod
```

## Conclusion

Podman fits naturally into Node.js development workflows. The critical detail to get right is the `node_modules` handling: use an anonymous volume to keep the container's dependencies separate from the host. Beyond that, the patterns are straightforward. Mount your source code, expose your ports, and use nodemon or your framework's built-in dev server for auto-reloading. For multi-container stacks with databases, `podman-compose` reads the same `docker-compose.yml` files you already know. The result is a development environment that works identically across every developer's machine.
