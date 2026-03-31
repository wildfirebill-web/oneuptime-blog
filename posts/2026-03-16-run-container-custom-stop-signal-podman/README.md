# How to Run a Container with Custom Stop Signal in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Container, DevOps, Signal, Process Management

Description: Learn how to configure custom stop signals for Podman containers to enable graceful shutdowns tailored to your application's signal handling.

---

> The right stop signal ensures your application shuts down gracefully, saving state and closing connections before exiting.

When you run `podman stop`, Podman sends a signal to the container's main process (PID 1) and waits for it to exit. By default, this signal is SIGTERM. However, some applications expect a different signal for graceful shutdown. For example, Nginx uses SIGQUIT for graceful shutdown, and some legacy applications use SIGUSR1 or SIGINT.

The `--stop-signal` flag lets you specify which signal Podman should send when stopping a container.

---

## Default Stop Behavior

By default, `podman stop` sends SIGTERM:

```bash
# Run a container that traps signals

podman run -d --name default-signal alpine sh -c "
  trap 'echo SIGTERM received; exit 0' TERM
  trap 'echo SIGINT received; exit 0' INT
  trap 'echo SIGQUIT received; exit 0' QUIT
  echo 'Waiting for signals...'
  while true; do sleep 1; done
"

# Stop sends SIGTERM by default
podman stop --time 5 default-signal
podman logs default-signal | tail -3

podman rm default-signal
```

## Setting a Custom Stop Signal

Use `--stop-signal` to specify a different signal:

```bash
# Use SIGINT as the stop signal
podman run -d --name sigint-app \
  --stop-signal SIGINT \
  alpine sh -c "
    trap 'echo SIGINT received - shutting down gracefully; exit 0' INT
    trap 'echo SIGTERM received' TERM
    echo 'Running... (expects SIGINT for shutdown)'
    while true; do sleep 1; done
  "

# When we stop, SIGINT is sent instead of SIGTERM
podman stop --time 5 sigint-app
podman logs sigint-app | tail -3

podman rm sigint-app
```

## Using Signal Numbers

You can use signal numbers instead of names:

```bash
# SIGINT is signal 2
podman run -d --name sig-number \
  --stop-signal 2 \
  alpine sh -c "
    trap 'echo Signal 2 (SIGINT) received; exit 0' INT
    while true; do sleep 1; done
  "

podman stop --time 5 sig-number
podman logs sig-number | tail -2
podman rm sig-number
```

## Common Stop Signals

```bash
# SIGTERM (15) - Default, standard termination
podman run -d --name use-sigterm --stop-signal SIGTERM alpine sleep infinity

# SIGINT (2) - Interrupt, like Ctrl+C
podman run -d --name use-sigint --stop-signal SIGINT alpine sleep infinity

# SIGQUIT (3) - Quit with core dump, used by Nginx for graceful shutdown
podman run -d --name use-sigquit --stop-signal SIGQUIT nginx:latest

# SIGUSR1 (10) - User-defined signal 1
podman run -d --name use-sigusr1 --stop-signal SIGUSR1 alpine sleep infinity

# SIGUSR2 (12) - User-defined signal 2
podman run -d --name use-sigusr2 --stop-signal SIGUSR2 alpine sleep infinity

# SIGHUP (1) - Hangup, often used for config reload
podman run -d --name use-sighup --stop-signal SIGHUP alpine sleep infinity

# Clean up all
podman stop use-sigterm use-sigint use-sigquit use-sigusr1 use-sigusr2 use-sighup 2>/dev/null
podman rm use-sigterm use-sigint use-sigquit use-sigusr1 use-sigusr2 use-sighup 2>/dev/null
```

## Nginx Graceful Shutdown Example

Nginx uses SIGQUIT for graceful shutdown (finish serving current requests):

```bash
# Run Nginx with SIGQUIT stop signal
podman run -d --name nginx-graceful \
  --stop-signal SIGQUIT \
  -p 8080:80 \
  nginx:latest

# When stopped, Nginx will finish serving current requests before exiting
podman stop --time 30 nginx-graceful

podman rm nginx-graceful
```

## Application with Custom Signal Handler

```bash
# Create a Python application that handles a custom signal
cat > /tmp/signal_app.py << 'EOF'
import signal
import sys
import time

def graceful_shutdown(signum, frame):
    print(f"Received signal {signum} - starting graceful shutdown")
    print("Closing database connections...")
    time.sleep(1)
    print("Flushing caches...")
    time.sleep(1)
    print("Shutdown complete")
    sys.exit(0)

# Register SIGUSR1 as the shutdown signal
signal.signal(signal.SIGUSR1, graceful_shutdown)

print("Application started, waiting for SIGUSR1 to shutdown...")
while True:
    time.sleep(1)
EOF

# Run with SIGUSR1 as the stop signal
podman run -d --name custom-app \
  --stop-signal SIGUSR1 \
  -v /tmp/signal_app.py:/app.py:Z \
  python:3.12-slim python3 /app.py

sleep 2

# Stop sends SIGUSR1, triggering the custom shutdown handler
podman stop --time 10 custom-app
podman logs custom-app

podman rm custom-app
```

## Verifying the Stop Signal

```bash
# Check what stop signal is configured
podman run -d --name check-signal \
  --stop-signal SIGQUIT \
  alpine sleep infinity

podman inspect check-signal --format '{{.Config.StopSignal}}'
# Output: 3 (SIGQUIT's number)

podman stop check-signal && podman rm check-signal
```

## Stop Signal with the --init Flag

When using `--init`, the init process forwards the stop signal to your application:

```bash
# Init properly forwards the custom stop signal
podman run -d --name init-signal \
  --init \
  --stop-signal SIGINT \
  alpine sh -c "
    trap 'echo SIGINT received via init; exit 0' INT
    while true; do sleep 1; done
  "

podman stop --time 5 init-signal
podman logs init-signal | tail -2
podman rm init-signal
```

## Summary

Custom stop signals in Podman ensure your applications shut down the way they expect:

- Use `--stop-signal` to specify which signal `podman stop` sends
- Default is SIGTERM (signal 15)
- Use signal names (SIGQUIT) or numbers (3)
- Nginx prefers SIGQUIT for graceful shutdown
- Custom applications can use SIGUSR1 or SIGUSR2
- Combine with `--init` for reliable signal forwarding
- Verify with `podman inspect` to confirm the configured signal

Match the stop signal to your application's signal handler for clean, predictable shutdowns.
