# How to Use SHELL Instruction in Containerfiles for Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containerfile, Shell, Container Build, DevOps

Description: Learn how to use the SHELL instruction in Containerfiles for Podman to change the default shell used for RUN, CMD, and ENTRYPOINT instructions in shell form.

---

> The SHELL instruction gives you control over which shell interprets your build commands, unlocking shell-specific features and enabling Windows container compatibility.

When you write a RUN instruction in shell form inside a Containerfile, Podman wraps it in a shell invocation. By default, that shell is `/bin/sh -c` on Linux. The SHELL instruction lets you change this default to any shell you prefer, whether that is bash for its richer feature set, ash for Alpine containers, or PowerShell for Windows containers. This guide covers how and when to use the SHELL instruction effectively with Podman.

---

## What Is the SHELL Instruction?

The SHELL instruction overrides the default shell used for the shell form of RUN, CMD, and ENTRYPOINT instructions. It does not affect the exec form (JSON array syntax), which bypasses the shell entirely.

The syntax is:

```dockerfile
SHELL ["executable", "parameters"]
```

The default on Linux is:

```dockerfile
SHELL ["/bin/sh", "-c"]
```

## Why Change the Default Shell?

The default `/bin/sh` on most Linux distributions is a minimal POSIX shell. It lacks features that bash and other shells provide, such as arrays, advanced string manipulation, pipefail, and extended globbing. If your build steps rely on these features, you need to either prefix every RUN instruction with `bash -c` or use the SHELL instruction to change the default.

Consider this common problem:

```dockerfile
FROM ubuntu:22.04

# This silently ignores failures in piped commands

RUN curl -s https://example.com/data | process-data > output.txt
```

If `curl` fails but `process-data` succeeds (because it receives empty input), the RUN instruction reports success. With bash's pipefail option, you can catch this:

```dockerfile
FROM ubuntu:22.04

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Now the RUN fails if curl fails, even in a pipeline
RUN curl -s https://example.com/data | process-data > output.txt
```

## Basic SHELL Usage

Here is a Containerfile that switches to bash for its build commands:

```dockerfile
FROM ubuntu:22.04

# Switch to bash with strict error handling
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Now we can use bash features like arrays
RUN declare -a packages=("vim" "git" "make") && \
    apt-get update && \
    apt-get install -y --no-install-recommends "${packages[@]}" && \
    rm -rf /var/lib/apt/lists/*

# Bash string manipulation
RUN version="v1.2.3" && \
    echo "Major version: ${version%%.*}" && \
    echo "Without prefix: ${version#v}"
```

Build with Podman:

```bash
podman build -t shell-demo .
```

## SHELL with Alpine Linux

Alpine Linux uses ash (part of BusyBox) as its default shell. If you need bash features on Alpine, install bash first and then change the shell:

```dockerfile
FROM alpine:3.19

# Install bash first (ash is the default)
RUN apk add --no-cache bash

# Switch to bash
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

# Now bash features are available
RUN source /etc/os-release && \
    echo "Running on ${PRETTY_NAME}" && \
    if [[ "${VERSION_ID}" > "3.18" ]]; then \
      echo "Modern Alpine detected"; \
    fi
```

Keep in mind that adding bash to an Alpine image increases its size. Only do this if you genuinely need bash features during the build process.

## Multiple SHELL Instructions

You can change the shell multiple times within a single Containerfile. Each SHELL instruction affects all subsequent shell-form instructions until the next SHELL instruction:

```dockerfile
FROM ubuntu:22.04

# Start with bash for complex build logic
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    && rm -rf /var/lib/apt/lists/*

# Switch to Python for inline scripting
SHELL ["/usr/bin/python3", "-c"]

RUN import platform; print(f"Python {platform.python_version()} on {platform.system()}")
RUN import os; [print(f"{k}={v}") for k, v in sorted(os.environ.items()) if k.startswith("PATH")]

# Switch back to bash for remaining setup
SHELL ["/bin/bash", "-c"]

RUN echo "Back to bash"
```

## Pipefail in Detail

The pipefail option is one of the most important reasons to use the SHELL instruction. By default, a pipeline's exit status is determined by the last command. With pipefail, the pipeline returns the exit status of the rightmost command that failed:

```dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Without pipefail: this succeeds even if curl fails
SHELL ["/bin/sh", "-c"]
RUN curl -sf https://nonexistent.example.com/file | cat > /dev/null || true

# With pipefail: this correctly fails if curl fails
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN curl -sf https://example.com/install.sh | bash
```

The Hadolint Containerfile linter actually flags piped RUN instructions that do not use pipefail as a warning (DL4006), which shows how widely this practice is recommended.

## SHELL for Build-Time Debugging

When debugging build failures, you can temporarily switch to a more verbose shell:

```dockerfile
FROM node:20-alpine

RUN apk add --no-cache bash

# Enable verbose mode for debugging
SHELL ["/bin/bash", "-euxo", "pipefail", "-c"]

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# Each command will be printed before execution due to -x flag
RUN node -e "console.log(process.version)" && \
    ls -la node_modules/.package-lock.json
```

The `-x` flag prints each command before execution, and `-u` treats unset variables as errors. Remove these flags for production builds to keep output clean.

## SHELL Interaction with ENTRYPOINT and CMD

The SHELL instruction only affects the shell form of ENTRYPOINT and CMD. The exec form is unaffected:

```dockerfile
FROM ubuntu:22.04

SHELL ["/bin/bash", "-c"]

# Shell form CMD - uses bash because of SHELL instruction
CMD echo "This runs in bash"

# Exec form CMD - runs directly, SHELL has no effect
CMD ["echo", "This runs directly, no shell"]
```

This distinction matters because the exec form is preferred for CMD and ENTRYPOINT in production. The SHELL instruction is primarily useful for RUN instructions during the build phase.

## Practical Example: Building a Go Application

Here is a real-world example that uses SHELL to build a Go application with strict error checking:

```dockerfile
FROM golang:1.22-alpine AS builder

RUN apk add --no-cache bash git

SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w -X main.version=$(git describe --tags --always)" \
    -o /server ./cmd/server

# Final stage does not need bash
FROM alpine:3.19

COPY --from=builder /server /server

ENTRYPOINT ["/server"]
```

Notice that the SHELL instruction is only used in the builder stage. The final stage uses the default shell because it only needs the exec form ENTRYPOINT.

## Best Practices

Use the SHELL instruction when you need bash-specific features like pipefail, arrays, or advanced string manipulation. Always enable pipefail when your RUN instructions contain pipes. In multi-stage builds, only change the shell in stages that need it. Remember that SHELL only affects shell form instructions, not exec form. Do not install bash in Alpine images just for the SHELL instruction unless you genuinely need bash features. Use `-euo pipefail` as your standard bash flags: `-e` exits on error, `-u` treats unset variables as errors, `-o pipefail` catches pipe failures. For production CMD and ENTRYPOINT, prefer the exec form which bypasses the shell entirely.

## Conclusion

The SHELL instruction is a targeted tool that solves specific problems around shell compatibility and error handling during container builds. Its most common and valuable use case is enabling bash's pipefail option to catch failures in piped commands. By understanding when the shell form is used versus the exec form, you can apply the SHELL instruction precisely where it adds value without adding unnecessary complexity to your Containerfiles.
