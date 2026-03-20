# How to Set Up a Node.js Development Environment with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Node.js, Development Environment, Docker, Hot Reload, JavaScript

Description: Learn how to set up a Node.js development environment with hot-reload and debugging in a Docker container managed by Portainer.

---

Containerizing your Node.js development environment with Portainer ensures consistent Node versions and dependencies across your team, with hot-reload for a productive development experience.

## Dev Environment Compose Stack

```yaml
version: "3.8"

services:
  node-dev:
    image: node:20-alpine
    restart: unless-stopped
    ports:
      - "3000:3000"    # Application
      - "9229:9229"    # Node.js debugger
    environment:
      NODE_ENV: development
    volumes:
      # Mount source code for hot-reload
      - ./src:/app
      # Use anonymous volume for node_modules to prevent host override
      - /app/node_modules
    working_dir: /app
    # Install dependencies and start with nodemon for hot-reload
    command: sh -c "npm install && npx nodemon --inspect=0.0.0.0:9229 server.js"
```

## Express Server with Hot-Reload

```javascript
// src/server.js
const express = require('express');
const app = express();

app.use(express.json());

app.get('/health', (req, res) => {
  res.json({ status: 'ok', node: process.version });
});

app.get('/', (req, res) => {
  // Edit this and save — nodemon picks it up
  res.json({ message: 'Node.js dev environment running' });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Package.json

```json
{
  "name": "node-dev-env",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.18.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.0",
    "jest": "^29.0.0"
  }
}
```

## Remote Debugging with VS Code

```json
// .vscode/launch.json
{
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to Container",
      "address": "localhost",
      "port": 9229,
      "localRoot": "${workspaceFolder}/src",
      "remoteRoot": "/app",
      "restart": true
    }
  ]
}
```

## Managing Multiple Node Versions

For projects requiring different Node versions, change the image tag in Portainer:

```yaml
# Node 18 LTS
image: node:18-alpine

# Node 20 LTS
image: node:20-alpine

# Latest Node
image: node:current-alpine
```

Update the image tag in the stack and redeploy via Portainer to switch versions without touching the host.
