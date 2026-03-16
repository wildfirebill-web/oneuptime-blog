# How to Run a Container with a Custom Command in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, CLI

Description: Learn how to override the default entrypoint and run custom commands inside Podman containers for flexible workload execution.

---

> Running custom commands in containers gives you precise control over what executes inside your containerized environment.

When you pull a container image, it comes with a default command or entrypoint defined by the image author. However, you often need to override that default behavior. Maybe you want to run a specific script, start a different process, or simply open an interactive shell for debugging. Podman makes this straightforward by letting you append any command to the `podman run` invocation.

This guide walks you through the various ways to run containers with custom commands in Podman, from simple overrides to advanced entrypoint manipulation.

---

## Basic Command Override

The simplest way to run a custom command is to append it after the image name:

```bash
# Run a simple echo command inside an Alpine container
podman run alpine echo "Hello from Podman"
```

This overrides whatever `CMD` was set in the Dockerfile. The container starts, runs the command, and exits.

## Running Interactive Commands

For interactive sessions, combine the `-it` flags with your custom command:

```bash
# Open a bash shell inside an Ubuntu container
podman run -it ubuntu /bin/bash

# Open a shell in Alpine (which uses ash/sh by default)
podman run -it alpine /bin/sh
```

The `-i` flag keeps STDIN open, and `-t` allocates a pseudo-TTY, making the session interactive.

## Running Multi-Part Commands with Shell

When your command includes pipes, redirections, or multiple statements, wrap it in a shell invocation:

```bash
# Run a pipeline inside the container
podman run alpine sh -c "cat /etc/os-release | head -5"

# Run multiple commands separated by semicolons
podman run alpine sh -c "echo 'Starting...'; sleep 2; echo 'Done!'"

# Use environment variables in the command
podman run alpine sh -c 'echo "Hostname: $(hostname), Date: $(date)"'
```

The `sh -c` pattern passes everything as a single string to the shell interpreter inside the container.

## Overriding the Entrypoint

Some images define an `ENTRYPOINT` instead of (or in addition to) `CMD`. To override the entrypoint entirely, use the `--entrypoint` flag:

```bash
# Override the entrypoint of an nginx image to run a shell
podman run --entrypoint /bin/bash -it nginx

# Override entrypoint and pass arguments
podman run --entrypoint echo nginx "Custom entrypoint output"
```

This is useful when an image has a rigid entrypoint that does not suit your needs.

## Combining Entrypoint and Command

You can set a custom entrypoint and pass arguments to it:

```bash
# Set python3 as the entrypoint and pass a script command
podman run --entrypoint python3 python:3.12-slim -c "print('Hello from Python in Podman')"

# Use entrypoint with multiple arguments
podman run --entrypoint /bin/sh alpine -c "uname -a && cat /etc/os-release"
```

## Running Commands in the Background

Combine a custom command with detached mode to run processes in the background:

```bash
# Run a long-lived process in the background
podman run -d --name my-worker alpine sh -c "while true; do echo 'Working...'; sleep 10; done"

# Check the logs to verify it is running
podman logs my-worker

# Stop and remove when done
podman stop my-worker
podman rm my-worker
```

## Running a Script from the Host

You can mount a local script into the container and execute it:

```bash
# Create a sample script on the host
cat > /tmp/my-script.sh << 'EOF'
#!/bin/sh
echo "Running custom script"
echo "Container OS: $(cat /etc/os-release | head -1)"
echo "Current date: $(date)"
echo "Disk usage:"
df -h /
EOF

chmod +x /tmp/my-script.sh

# Mount and execute the script inside the container
podman run -v /tmp/my-script.sh:/usr/local/bin/my-script.sh:Z alpine /usr/local/bin/my-script.sh
```

## Passing Environment Variables to Custom Commands

Environment variables can be used alongside custom commands:

```bash
# Pass environment variables that the command can use
podman run -e MY_VAR="Podman" alpine sh -c 'echo "Hello from $MY_VAR"'

# Use an env file with a custom command
cat > /tmp/env.list << 'EOF'
APP_NAME=MyApp
APP_VERSION=1.0
EOF

podman run --env-file /tmp/env.list alpine sh -c 'echo "$APP_NAME version $APP_VERSION"'
```

## Setting the Working Directory for Commands

Control where your custom command executes with `--workdir`:

```bash
# Run a command in a specific directory
podman run --workdir /tmp alpine sh -c "pwd && ls -la"

# Combine with volume mounts
podman run --workdir /app -v ./myproject:/app:Z node:20-slim sh -c "ls -la && node --version"
```

## Specifying the User for Command Execution

Run your custom command as a specific user:

```bash
# Run the command as the nobody user
podman run --user nobody alpine sh -c "whoami && id"

# Run as a specific UID:GID
podman run --user 1000:1000 alpine sh -c "id"
```

## Practical Example: Database Initialization

A common real-world use case is running database utilities:

```bash
# Run a one-off PostgreSQL command to create a database dump
podman run --rm \
  -e PGPASSWORD=mysecretpassword \
  postgres:16 \
  pg_dump -h host.containers.internal -U postgres -d mydb > backup.sql

# Run a MySQL client with a custom query
podman run --rm \
  mysql:8 \
  mysql -h host.containers.internal -u root -pmypassword -e "SHOW DATABASES;"
```

## Summary

Running containers with custom commands in Podman is essential for flexible container usage. The key patterns are:

- Append a command after the image name to override `CMD`
- Use `sh -c "..."` for complex commands with pipes or chaining
- Use `--entrypoint` to override the image entrypoint
- Combine with `-it` for interactive sessions or `-d` for background execution
- Mount scripts from the host for more complex workflows

These techniques let you treat containers as lightweight, disposable execution environments for any task.
