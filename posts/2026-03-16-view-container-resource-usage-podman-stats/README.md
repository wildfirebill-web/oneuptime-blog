# How to View Container Resource Usage with podman stats

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Monitoring, Resource Usage

Description: Learn how to monitor CPU, memory, network, and disk I/O usage of Podman containers using the podman stats command.

---

> Monitoring container resource usage is critical for performance tuning, capacity planning, and identifying resource-hungry containers.

Knowing how much CPU, memory, and I/O your containers consume helps you right-size resource limits, spot performance problems, and prevent resource exhaustion. The `podman stats` command provides a real-time view of resource consumption for all running containers. This guide covers everything you need to know.

---

## Basic Usage

View resource usage for all running containers:

```bash
# Start some test containers

podman run -d --name web nginx:latest
podman run -d --name db -e POSTGRES_PASSWORD=secret postgres:latest

# View stats for all running containers (one-time snapshot)
podman stats --no-stream
```

Output columns include:
- **ID** - Container ID
- **NAME** - Container name
- **CPU %** - CPU usage percentage
- **MEM USAGE / LIMIT** - Memory used vs. available
- **MEM %** - Memory usage percentage
- **NET I/O** - Network bytes in/out
- **BLOCK I/O** - Disk read/write bytes
- **PIDS** - Number of processes

## Stats for Specific Containers

Monitor only the containers you care about:

```bash
# Stats for a single container
podman stats --no-stream web

# Stats for multiple specific containers
podman stats --no-stream web db

# Stats by container ID
podman stats --no-stream $(podman ps -q)
```

## Custom Output Format

Customize the output with `--format`:

```bash
# Custom table format
podman stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"

# JSON output
podman stats --no-stream --format json

# Just names and CPU usage
podman stats --no-stream --format "{{.Name}}: CPU={{.CPUPerc}}, MEM={{.MemPerc}}"
```

Available format fields:
- `.ID` - Container ID
- `.Name` - Container name
- `.CPUPerc` - CPU percentage
- `.MemUsage` - Memory usage and limit
- `.MemPerc` - Memory percentage
- `.NetIO` - Network I/O
- `.BlockIO` - Block I/O
- `.PIDs` - Number of PIDs

## Monitoring with Resource Limits

Set resource limits and monitor compliance:

```bash
# Run a container with memory limit
podman run -d --name limited --memory 128m nginx:latest

# Check how close it is to the limit
podman stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}" limited
```

## Scripting with Stats

Use stats output in monitoring scripts:

```bash
# Alert if any container exceeds 80% memory
podman stats --no-stream --format "{{.Name}} {{.MemPerc}}" | while read name mem_pct; do
    # Remove the % sign for comparison
    mem_num=$(echo "$mem_pct" | tr -d '%')
    if [ "$(echo "$mem_num > 80" | bc -l 2>/dev/null)" = "1" ]; then
        echo "WARNING: $name is using ${mem_pct} memory"
    fi
done

# Log stats to a file periodically
podman stats --no-stream --format "{{.Name}},{{.CPUPerc}},{{.MemPerc}},{{.NetIO}}" >> /tmp/container-stats.csv
```

## Comparing Container Resource Usage

Generate a resource comparison report:

```bash
# Resource usage summary
echo "=== Container Resource Report ==="
echo "Date: $(date)"
echo ""
podman stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.PIDs}}"
echo ""
echo "Total containers: $(podman ps -q | wc -l)"
```

## Understanding the Numbers

```bash
# Detailed breakdown
podman stats --no-stream --format "
Container: {{.Name}}
  CPU:     {{.CPUPerc}} (percentage of total CPU)
  Memory:  {{.MemUsage}} (used / limit)
  Memory:  {{.MemPerc}} (percentage of limit)
  Net I/O: {{.NetIO}} (received / sent)
  Disk:    {{.BlockIO}} (read / written)
  PIDs:    {{.PIDs}} (number of processes)
" web
```

## Cleanup

```bash
podman stop web db limited 2>/dev/null
podman rm web db limited 2>/dev/null
```

## Summary

The `podman stats` command is your go-to tool for monitoring container resource consumption. Use `--no-stream` for one-time snapshots, `--format` for custom output, and combine it with shell scripting for automated monitoring and alerting. Regular monitoring helps you optimize resource allocation and catch problems early.
