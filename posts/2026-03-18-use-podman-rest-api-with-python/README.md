# How to Use the Podman REST API with Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, REST API, Python, Container, Automation

Description: Learn how to interact with the Podman REST API using Python, including managing containers, images, and volumes with the requests library and building reusable client classes.

---

> Python is one of the most popular languages for infrastructure automation and DevOps tooling. By combining Python with the Podman REST API, you can build powerful container management scripts, monitoring tools, and deployment automation that integrates naturally with your existing Python ecosystem.

Python's rich library ecosystem, clean syntax, and strong HTTP client support make it an excellent choice for working with the Podman REST API. Whether you are writing a deployment script, building a custom dashboard, or creating an automated testing framework that manages containers, Python provides the tools you need.

This guide walks through using the `requests` library (with Unix socket support) to interact with the Podman REST API, covering container lifecycle management, image operations, and building a reusable client class.

---

## Setting Up the Environment

First, install the required packages:

```bash
pip install requests requests-unixsocket urllib3
```

The `requests-unixsocket` package adds Unix domain socket support to the standard `requests` library.

## Connecting to the Podman Socket

Here is the basic pattern for connecting to the Podman API through a Unix socket:

```python
import requests_unixsocket
import json

session = requests_unixsocket.Session()

# Unix socket paths must be URL-encoded

# /run/podman/podman.sock becomes %2Frun%2Fpodman%2Fpodman.sock
SOCKET_PATH = "%2Frun%2Fpodman%2Fpodman.sock"
BASE_URL = f"http+unix://{SOCKET_PATH}/v4.0.0/libpod"

# Test connectivity
response = session.get(f"{BASE_URL}/info")
info = response.json()
print(f"Connected to Podman on {info['host']['hostname']}")
print(f"Podman version: {info['version']['Version']}")
```

## Building a Podman Client Class

To keep your code organized, wrap the API interactions in a client class:

```python
import requests_unixsocket
import json
import time
from urllib.parse import quote


class PodmanClient:
    def __init__(self, socket_path="/run/podman/podman.sock", api_version="v4.0.0"):
        self.session = requests_unixsocket.Session()
        encoded_path = quote(socket_path, safe="")
        self.base_url = f"http+unix://{encoded_path}/{api_version}/libpod"

    def _request(self, method, endpoint, **kwargs):
        url = f"{self.base_url}{endpoint}"
        response = self.session.request(method, url, **kwargs)
        response.raise_for_status()
        if response.content:
            return response.json()
        return None

    def get(self, endpoint, **kwargs):
        return self._request("GET", endpoint, **kwargs)

    def post(self, endpoint, **kwargs):
        return self._request("POST", endpoint, **kwargs)

    def delete(self, endpoint, **kwargs):
        return self._request("DELETE", endpoint, **kwargs)

    # System
    def info(self):
        return self.get("/info")

    def version(self):
        return self.get("/version")

    # Containers
    def list_containers(self, all_containers=False):
        return self.get("/containers/json", params={"all": all_containers})

    def create_container(self, spec):
        return self.post("/containers/create", json=spec)

    def start_container(self, name):
        return self.post(f"/containers/{name}/start")

    def stop_container(self, name, timeout=10):
        return self.post(f"/containers/{name}/stop", params={"timeout": timeout})

    def remove_container(self, name, force=False):
        return self.delete(f"/containers/{name}", params={"force": force})

    def inspect_container(self, name):
        return self.get(f"/containers/{name}/json")

    def container_logs(self, name, tail=100, timestamps=False):
        url = f"{self.base_url}/containers/{name}/logs"
        response = self.session.get(url, params={
            "stdout": True,
            "stderr": True,
            "tail": tail,
            "timestamps": timestamps
        })
        return response.text

    # Images
    def list_images(self):
        return self.get("/images/json")

    def pull_image(self, reference):
        return self.post("/images/pull", params={"reference": reference})

    def remove_image(self, name, force=False):
        return self.delete(f"/images/{name}", params={"force": force})

    # Volumes
    def list_volumes(self):
        return self.get("/volumes/json")

    def create_volume(self, name, labels=None):
        spec = {"Name": name}
        if labels:
            spec["Labels"] = labels
        return self.post("/volumes/create", json=spec)

    def remove_volume(self, name, force=False):
        return self.delete(f"/volumes/{name}", params={"force": force})
```

## Managing Container Lifecycle

Use the client class to manage the full container lifecycle:

```python
client = PodmanClient()

# Create and start a container
container_spec = {
    "image": "docker.io/library/nginx:alpine",
    "name": "python-demo",
    "portmappings": [
        {
            "container_port": 80,
            "host_port": 8080,
            "protocol": "tcp"
        }
    ],
    "env": {
        "NGINX_HOST": "localhost"
    }
}

result = client.create_container(container_spec)
print(f"Container created: {result['Id'][:12]}")

client.start_container("python-demo")
print("Container started")

# Inspect the container
info = client.inspect_container("python-demo")
print(f"Status: {info['State']['Status']}")
print(f"PID: {info['State']['Pid']}")

# Get logs
logs = client.container_logs("python-demo", tail=10)
print(f"Recent logs:\n{logs}")

# Stop and remove
client.stop_container("python-demo", timeout=5)
client.remove_container("python-demo")
print("Container removed")
```

## Monitoring Container Stats

Retrieve and process container statistics:

```python
import requests_unixsocket
from urllib.parse import quote


def get_container_stats(socket_path, container_name):
    session = requests_unixsocket.Session()
    encoded = quote(socket_path, safe="")
    url = f"http+unix://{encoded}/v4.0.0/libpod/containers/{container_name}/stats"

    response = session.get(url, params={"stream": False}, stream=True)

    for line in response.iter_lines():
        if line:
            stats = json.loads(line)
            return {
                "name": stats.get("Name"),
                "cpu_percent": round(stats.get("CPU", 0), 2),
                "memory_mb": round(stats.get("MemUsage", 0) / 1048576, 2),
                "memory_limit_mb": round(stats.get("MemLimit", 0) / 1048576, 2),
                "memory_percent": round(stats.get("MemPerc", 0), 2),
                "net_input_mb": round(stats.get("NetInput", 0) / 1048576, 2),
                "net_output_mb": round(stats.get("NetOutput", 0) / 1048576, 2),
                "pids": stats.get("PIDs", 0),
            }


stats = get_container_stats("/run/podman/podman.sock", "my-container")
print(json.dumps(stats, indent=2))
```

## Executing Commands in Containers

Use the exec API to run commands inside containers:

```python
def exec_in_container(client, container_name, command, user=None):
    """Execute a command in a container and return the output."""
    session = client.session
    base = client.base_url

    # Step 1: Create exec instance
    exec_config = {
        "AttachStdout": True,
        "AttachStderr": True,
        "Cmd": command if isinstance(command, list) else command.split()
    }
    if user:
        exec_config["User"] = user

    response = session.post(f"{base}/containers/{container_name}/exec", json=exec_config)
    response.raise_for_status()
    exec_id = response.json()["Id"]

    # Step 2: Start exec instance
    response = session.post(f"{base}/exec/{exec_id}/start", json={"Detach": False})
    output = response.text

    # Step 3: Get exit code
    response = session.get(f"{base}/exec/{exec_id}/json")
    exit_code = response.json().get("ExitCode", -1)

    return {"output": output, "exit_code": exit_code}


client = PodmanClient()

result = exec_in_container(client, "my-container", ["cat", "/etc/os-release"])
print(f"Exit code: {result['exit_code']}")
print(f"Output:\n{result['output']}")

result = exec_in_container(client, "my-container", "df -h", user="nobody")
print(f"Disk usage:\n{result['output']}")
```

## Image Management

Work with container images programmatically:

```python
client = PodmanClient()

# List all images
images = client.list_images()
for img in images:
    names = img.get("Names") or ["<none>"]
    size_mb = img["Size"] / 1048576
    print(f"  {names[0]} ({size_mb:.1f} MB)")

# Pull an image
print("Pulling alpine:latest...")
client.pull_image("docker.io/library/alpine:latest")
print("Done")

# Clean up unused images
def prune_images(client):
    """Remove dangling images."""
    images = client.list_images()
    removed = 0
    for img in images:
        if not img.get("Names") or img["Names"] == [None]:
            try:
                client.remove_image(img["Id"][:12])
                removed += 1
            except Exception as e:
                print(f"  Could not remove {img['Id'][:12]}: {e}")
    print(f"Removed {removed} dangling images")

prune_images(client)
```

## Error Handling

Implement proper error handling for production scripts:

```python
from requests.exceptions import ConnectionError, HTTPError


class PodmanAPIError(Exception):
    def __init__(self, status_code, message):
        self.status_code = status_code
        self.message = message
        super().__init__(f"HTTP {status_code}: {message}")


def safe_api_call(func, *args, **kwargs):
    """Wrapper for API calls with error handling."""
    try:
        return func(*args, **kwargs)
    except ConnectionError:
        print("ERROR: Cannot connect to Podman API. Is the service running?")
        return None
    except HTTPError as e:
        status = e.response.status_code
        try:
            detail = e.response.json().get("message", str(e))
        except ValueError:
            detail = e.response.text
        raise PodmanAPIError(status, detail)


client = PodmanClient()

try:
    info = safe_api_call(client.inspect_container, "nonexistent")
except PodmanAPIError as e:
    if e.status_code == 404:
        print("Container not found")
    else:
        print(f"API error: {e}")
```

## Streaming Logs with Python

Stream container logs in real time:

```python
def stream_logs(socket_path, container_name, callback=None):
    """Stream container logs and process each line."""
    session = requests_unixsocket.Session()
    encoded = quote(socket_path, safe="")
    url = f"http+unix://{encoded}/v4.0.0/libpod/containers/{container_name}/logs"

    response = session.get(url, params={
        "follow": True,
        "stdout": True,
        "stderr": True,
        "timestamps": True
    }, stream=True)

    try:
        for line in response.iter_lines():
            if line:
                decoded = line.decode("utf-8", errors="replace")
                if callback:
                    callback(decoded)
                else:
                    print(decoded)
    except KeyboardInterrupt:
        print("\nStopped streaming")


# Usage
stream_logs("/run/podman/podman.sock", "my-container")
```

## Batch Operations

Perform operations across multiple containers:

```python
def restart_all_containers(client):
    """Restart all running containers."""
    containers = client.list_containers(all_containers=False)
    for container in containers:
        name = container["Names"][0]
        print(f"Restarting {name}...")
        try:
            client.stop_container(name, timeout=10)
            client.start_container(name)
            print(f"  {name} restarted")
        except Exception as e:
            print(f"  Failed to restart {name}: {e}")


def cleanup_stopped_containers(client):
    """Remove all stopped containers."""
    containers = client.list_containers(all_containers=True)
    removed = 0
    for container in containers:
        if container["State"] == "exited":
            name = container["Names"][0]
            try:
                client.remove_container(name)
                removed += 1
            except Exception as e:
                print(f"Could not remove {name}: {e}")
    print(f"Removed {removed} stopped containers")
```

## Conclusion

Python combined with the Podman REST API provides a powerful platform for container automation. The `requests-unixsocket` library makes it straightforward to communicate with the Podman socket, and wrapping the API in a client class creates a clean, reusable interface. From simple scripts to complex orchestration workflows, Python gives you the flexibility to build container management tools that integrate seamlessly with your infrastructure automation stack.
