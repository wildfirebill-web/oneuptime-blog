# How to Create Containers with Podman Python SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Python, SDK, Containers, Create

Description: A comprehensive guide to creating, configuring, and running containers programmatically using the Podman Python SDK, including port mapping, volume mounts, environment variables, and resource limits.

---

> Creating containers programmatically gives you repeatable, version-controlled infrastructure. The Podman Python SDK lets you define every aspect of a container in clean Python code.

Moving from manual `podman run` commands to programmatic container creation brings consistency and automation to your workflows. The Podman Python SDK provides a rich API for creating containers with full control over networking, storage, environment, and resource allocation. This guide walks through every configuration option you need.

---

## Creating a Simple Container

The most basic container creation requires only an image name:

```python
from podman import PodmanClient

with PodmanClient() as client:
    # Pull the image first
    client.images.pull("docker.io/library/nginx:latest")

    # Create a container
    container = client.containers.create(
        image="nginx:latest",
        name="my-nginx"
    )

    print(f"Created: {container.name} ({container.short_id})")
    print(f"Status: {container.status}")
```

The container is created but not started. Its status will be "created" until you explicitly start it.

## Creating and Starting a Container

To create and immediately start a container, use `containers.run()`:

```python
from podman import PodmanClient

with PodmanClient() as client:
    container = client.containers.run(
        image="nginx:latest",
        name="web-server",
        detach=True
    )

    print(f"Running: {container.name}")
    print(f"Status: {container.status}")
```

The `detach=True` parameter runs the container in the background, similar to `podman run -d`.

## Configuring Port Mappings

Map container ports to host ports for network access:

```python
from podman import PodmanClient

with PodmanClient() as client:
    container = client.containers.run(
        image="nginx:latest",
        name="web-with-ports",
        detach=True,
        ports={
            "80/tcp": 8080,      # Container port 80 -> Host port 8080
            "443/tcp": 8443      # Container port 443 -> Host port 8443
        }
    )

    print(f"Ports: {container.ports}")
```

For binding to a specific host interface:

```python
from podman import PodmanClient

with PodmanClient() as client:
    container = client.containers.run(
        image="nginx:latest",
        name="web-specific-ip",
        detach=True,
        ports={
            "80/tcp": ("127.0.0.1", 8080),  # Bind to localhost only
        }
    )
```

To map a container port to a random host port:

```python
from podman import PodmanClient

with PodmanClient() as client:
    container = client.containers.run(
        image="nginx:latest",
        name="web-random-port",
        detach=True,
        ports={"80/tcp": None}  # Random host port
    )

    # Reload to get assigned port
    container.reload()
    print(f"Assigned ports: {container.ports}")
```

## Setting Environment Variables

Pass configuration to containers through environment variables:

```python
from podman import PodmanClient

with PodmanClient() as client:
    container = client.containers.run(
        image="postgres:15",
        name="my-database",
        detach=True,
        environment={
            "POSTGRES_USER": "admin",
            "POSTGRES_PASSWORD": "secretpass",
            "POSTGRES_DB": "myapp",
            "PGDATA": "/var/lib/postgresql/data/pgdata"
        }
    )

    print(f"Database container: {container.name}")
```

You can also pass environment variables as a list:

```python
environment = [
    "POSTGRES_USER=admin",
    "POSTGRES_PASSWORD=secretpass",
    "POSTGRES_DB=myapp"
]
```

## Mounting Volumes

Bind mount host directories or use named volumes:

```python
from podman import PodmanClient

with PodmanClient() as client:
    # Bind mount a host directory
    container = client.containers.run(
        image="nginx:latest",
        name="web-with-content",
        detach=True,
        volumes={
            "/home/user/website": {
                "bind": "/usr/share/nginx/html",
                "mode": "ro"  # Read-only
            }
        }
    )

    print(f"Mounted volumes: {container.attrs['Mounts']}")
```

Using named volumes:

```python
from podman import PodmanClient

with PodmanClient() as client:
    # Create a named volume first
    volume = client.volumes.create(name="db-data")

    # Use it in a container
    container = client.containers.run(
        image="postgres:15",
        name="db-with-volume",
        detach=True,
        environment={"POSTGRES_PASSWORD": "secret"},
        volumes={
            "db-data": {
                "bind": "/var/lib/postgresql/data",
                "mode": "rw"
            }
        }
    )
```

## Setting Resource Limits

Control CPU and memory allocation:

```python
from podman import PodmanClient

with PodmanClient() as client:
    container = client.containers.run(
        image="python:3.11",
        name="limited-app",
        detach=True,
        command=["python3", "-c", "import time; time.sleep(3600)"],
        mem_limit="512m",        # 512 MB memory limit
        cpu_period=100000,       # CPU period in microseconds
        cpu_quota=50000,         # 50% of one CPU
        memswap_limit="1g",      # Swap limit
    )

    print(f"Container: {container.name}")
    host_config = container.attrs.get("HostConfig", {})
    print(f"Memory limit: {host_config.get('Memory')}")
```

## Running Commands in Containers

Create containers that run specific commands:

```python
from podman import PodmanClient

with PodmanClient() as client:
    # Run a one-shot command
    container = client.containers.run(
        image="python:3.11",
        name="python-script",
        command=["python3", "-c", "print('Hello from Podman!')"],
        detach=True
    )

    # Wait for completion and get logs
    container.wait()
    output = container.logs()
    print(f"Output: {output.decode()}")
```

For interactive-style containers with a custom entrypoint:

```python
from podman import PodmanClient

with PodmanClient() as client:
    container = client.containers.run(
        image="alpine:latest",
        name="custom-entry",
        entrypoint=["/bin/sh"],
        command=["-c", "echo 'System info:' && uname -a && cat /etc/os-release"],
        detach=True
    )

    container.wait()
    print(container.logs().decode())
```

## Adding Labels

Labels help organize and query containers:

```python
from podman import PodmanClient

with PodmanClient() as client:
    container = client.containers.run(
        image="nginx:latest",
        name="labeled-web",
        detach=True,
        labels={
            "app": "web-frontend",
            "environment": "staging",
            "team": "platform",
            "version": "2.1.0"
        }
    )

    print(f"Labels: {container.labels}")
```

## Configuring Networking

Attach containers to specific networks:

```python
from podman import PodmanClient

with PodmanClient() as client:
    # Create a custom network
    network = client.networks.create(
        name="app-network",
        driver="bridge"
    )

    # Create container on the custom network
    container = client.containers.run(
        image="nginx:latest",
        name="web-on-network",
        detach=True,
        network=network.name
    )

    print(f"Network: {container.attrs['NetworkSettings']}")
```

## Container Restart Policies

Configure automatic restart behavior:

```python
from podman import PodmanClient

with PodmanClient() as client:
    container = client.containers.run(
        image="nginx:latest",
        name="auto-restart-web",
        detach=True,
        restart_policy={
            "Name": "on-failure",
            "MaximumRetryCount": 5
        }
    )
```

Available policies include `no`, `always`, `on-failure`, and `unless-stopped`.

## Managing Container Lifecycle

After creating a container, manage its full lifecycle:

```python
from podman import PodmanClient
import time

with PodmanClient() as client:
    # Create
    container = client.containers.run(
        image="nginx:latest",
        name="lifecycle-demo",
        detach=True
    )
    print(f"Started: {container.status}")

    # Pause
    container.pause()
    container.reload()
    print(f"Paused: {container.status}")

    # Unpause
    container.unpause()
    container.reload()
    print(f"Unpaused: {container.status}")

    # Stop
    container.stop(timeout=10)
    container.reload()
    print(f"Stopped: {container.status}")

    # Restart
    container.restart()
    container.reload()
    print(f"Restarted: {container.status}")

    # Remove (must stop first)
    container.stop()
    container.remove()
    print("Removed")
```

## Creating Multiple Related Containers

Build a multi-container application:

```python
from podman import PodmanClient

def create_web_stack():
    """Create a web application stack with database and web server."""
    with PodmanClient() as client:
        # Create network
        network = client.networks.create(name="web-stack")

        # Database container
        db = client.containers.run(
            image="postgres:15",
            name="stack-db",
            detach=True,
            network=network.name,
            environment={
                "POSTGRES_PASSWORD": "dbpass",
                "POSTGRES_DB": "webapp"
            },
            volumes={"stack-db-data": {"bind": "/var/lib/postgresql/data", "mode": "rw"}}
        )
        print(f"Database: {db.name} ({db.status})")

        # Redis container
        cache = client.containers.run(
            image="redis:7",
            name="stack-cache",
            detach=True,
            network=network.name
        )
        print(f"Cache: {cache.name} ({cache.status})")

        # Web server container
        web = client.containers.run(
            image="nginx:latest",
            name="stack-web",
            detach=True,
            network=network.name,
            ports={"80/tcp": 8080},
            environment={
                "DB_HOST": "stack-db",
                "CACHE_HOST": "stack-cache"
            }
        )
        print(f"Web: {web.name} ({web.status})")

        return [db, cache, web]

containers = create_web_stack()
```

## Conclusion

The Podman Python SDK gives you complete control over container creation, from simple single-container setups to complex multi-container application stacks. By defining containers in Python code, you get repeatable deployments, version control, and the ability to integrate container management into larger automation workflows. The next step is learning how to manage the images that your containers depend on.
