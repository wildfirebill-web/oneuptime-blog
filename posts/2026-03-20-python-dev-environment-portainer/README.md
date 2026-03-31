# How to Set Up a Python Development Environment with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Python, Development Environment, Docker, Virtual Environments, Dev Container

Description: Learn how to set up a Python development environment with hot-reload and debugging support in a container managed by Portainer.

---

Running your Python development environment in Docker via Portainer ensures all team members use identical dependencies and eliminates "works on my machine" issues. This guide creates a Python dev container with hot-reload.

## Dev Environment Compose Stack

```yaml
version: "3.8"

services:
  python-dev:
    image: python:3.12-slim
    restart: unless-stopped
    ports:
      - "8000:8000"    # Application
      - "5678:5678"    # debugpy remote debugger
    environment:
      PYTHONDONTWRITEBYTECODE: "1"
      PYTHONUNBUFFERED: "1"
    volumes:
      # Mount your source code for hot-reload
      - ./src:/app
      - pip_cache:/root/.cache/pip    # Cache pip downloads
    working_dir: /app
    command: >
      sh -c "
        pip install -r requirements.txt &&
        python -m debugpy --listen 0.0.0.0:5678 -m uvicorn main:app --host 0.0.0.0 --port 8000 --reload
      "

volumes:
  pip_cache:
```

## Requirements

```text
# src/requirements.txt

fastapi
uvicorn[standard]
debugpy
httpx
pytest
pytest-asyncio
```

## Hot-Reload Development

The `--reload` flag in uvicorn watches for file changes. When you edit code in `./src`, the development server automatically restarts:

```python
# src/main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    # Edit this and save - hot-reload picks it up automatically
    return {"message": "Hello, Python dev environment!"}
```

## Remote Debugging with VS Code

Attach VS Code to the running container's debugpy listener:

```json
// .vscode/launch.json
{
  "configurations": [
    {
      "name": "Python: Remote Attach",
      "type": "python",
      "request": "attach",
      "connect": {
        "host": "localhost",
        "port": 5678
      },
      "pathMappings": [
        {
          "localRoot": "${workspaceFolder}/src",
          "remoteRoot": "/app"
        }
      ]
    }
  ]
}
```

## Running Tests in the Container

```bash
# Via Portainer Exec console or docker exec:
cd /app && pytest tests/ -v

# Or run a specific test file
pytest tests/test_api.py -v
```
