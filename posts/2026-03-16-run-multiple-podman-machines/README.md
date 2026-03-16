# How to Run Multiple Podman Machines Simultaneously

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Podman Machine, Virtual Machine, Containers, DevOps

Description: Learn how to create, start, and manage multiple Podman machines running at the same time for isolated development environments.

---

> Running multiple Podman machines lets you maintain isolated environments for different projects, testing scenarios, or team configurations.

There are many reasons to run more than one Podman machine. You might need separate environments for different projects, want to test across different configurations, or need isolated machines for CI/CD pipelines. This guide shows you how to create and manage multiple Podman machines running simultaneously.

---

## Creating Multiple Machines

Start by initializing several machines with different configurations.

```bash
# Create a machine for web development
podman machine init web-dev --cpus 2 --memory 4096 --disk-size 50

# Create a machine for database work
podman machine init db-dev --cpus 4 --memory 8192 --disk-size 200

# Create a lightweight machine for testing
podman machine init test-env --cpus 1 --memory 2048 --disk-size 30
```

## Starting Multiple Machines

Start each machine individually. They will run simultaneously.

```bash
# Start all three machines
podman machine start web-dev
podman machine start db-dev
podman machine start test-env
```

You can verify they are all running:

```bash
# Check status of all machines
podman machine ls
```

The output shows each machine and its state:

```
NAME        VM TYPE     CREATED        LAST UP            CPUS    MEMORY      DISK SIZE
web-dev*    qemu        1 minute ago   Currently running  2       4.295GB     53.69GB
db-dev      qemu        1 minute ago   Currently running  4       8.59GB      214.7GB
test-env    qemu        1 minute ago   Currently running  1       2.147GB     32.21GB
```

## Starting All Machines with a Script

Automate starting all machines at once:

```bash
# Start all existing machines
podman machine ls --format "{{.Name}}" | while read -r machine; do
    echo "Starting $machine..."
    podman machine start "$machine" 2>/dev/null
done
```

## Running Containers on Specific Machines

Use the `--connection` flag to target a specific machine when running containers.

```bash
# Run nginx on the web-dev machine
podman --connection web-dev run -d --name web-server -p 8080:80 nginx

# Run PostgreSQL on the db-dev machine
podman --connection db-dev run -d --name postgres -p 5432:5432 \
    -e POSTGRES_PASSWORD=mysecret postgres:16

# Run a test container on the test-env machine
podman --connection test-env run -d --name test-app alpine sleep 3600
```

## Listing Connections

Podman system connections map to machines. List them to see available connections.

```bash
# List all system connections
podman system connection ls
```

Output:

```
Name        URI                                                         Identity                    Default
web-dev     ssh://core@localhost:54321/run/podman/podman.sock          /home/user/.ssh/podman-rsa  true
db-dev      ssh://core@localhost:54322/run/podman/podman.sock          /home/user/.ssh/podman-rsa  false
test-env    ssh://core@localhost:54323/run/podman/podman.sock          /home/user/.ssh/podman-rsa  false
```

## Managing Resources Across Machines

Keep track of total resource usage across all running machines:

```bash
# Show combined resource allocation
echo "=== Resource Allocation ==="
total_cpus=0
total_memory=0

podman machine ls --format json | jq -r '.[] | select(.Running == true) | .Name' | while read -r machine; do
    cpus=$(podman machine inspect "$machine" | jq '.Resources.CPUs')
    memory=$(podman machine inspect "$machine" | jq '.Resources.Memory')
    echo "$machine: ${cpus} CPUs, ${memory} MB RAM"
done

# Total resources
podman machine ls --format json | jq '[.[] | select(.Running == true)] | {
    total_cpus: (map(.CPUs) | add),
    total_memory_mb: (map(.Memory) | add)
}'
```

## Stopping Specific Machines

Stop individual machines when you no longer need them:

```bash
# Stop just the test environment
podman machine stop test-env

# Stop all machines
podman machine ls --format "{{.Name}}" | while read -r machine; do
    podman machine stop "$machine" 2>/dev/null
done
```

## Platform Considerations

Be aware of resource constraints when running multiple machines:

```bash
# Check your system resources before creating machines (macOS)
sysctl -n hw.ncpu          # Total CPU cores
sysctl -n hw.memsize       # Total memory in bytes

# Recommended: Do not allocate more than 70-80% of host resources
# across all machines combined
```

On macOS, each machine uses the Apple Virtualization Framework or QEMU, and each has its own resource allocation. On Linux with WSL, resource sharing works differently.

## Quick Reference

| Command | Purpose |
|---|---|
| `podman machine init <name>` | Create a new machine |
| `podman machine start <name>` | Start a specific machine |
| `podman --connection <name> run ...` | Run a container on a specific machine |
| `podman system connection ls` | List all machine connections |
| `podman machine stop <name>` | Stop a specific machine |

## Summary

Running multiple Podman machines simultaneously gives you isolated environments for different workloads. Create machines with appropriate resource allocations, use `--connection` to target specific machines, and monitor total resource usage to avoid overcommitting your host system. This approach is excellent for maintaining separate development, testing, and staging environments on a single workstation.
