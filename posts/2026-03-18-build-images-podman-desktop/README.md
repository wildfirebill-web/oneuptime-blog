# How to Build Images with Podman Desktop

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Podman Desktop, Image Building, Containerfile

Description: Learn how to build container images using Podman Desktop's graphical interface and the equivalent CLI commands.

---

> Building container images with Podman Desktop gives you a visual workflow for creating, tagging, and managing images without leaving the graphical interface.

Building container images is a fundamental task in container development. Podman Desktop provides a graphical interface for building images from Containerfiles (or Dockerfiles), making it accessible for developers who prefer a visual workflow. This post covers building images through both Podman Desktop and the CLI, including multi-stage builds and build arguments.

---

## Prerequisites

Ensure Podman Desktop is installed and the Podman engine is running.

```bash
# Verify Podman is running

podman info

# Check the machine status (macOS/Windows)
podman machine list
```

## Creating a Containerfile

Start by writing a Containerfile for your application.

```bash
# Create a project directory
mkdir -p ~/my-web-app
cd ~/my-web-app

# Create a simple web application
cat > index.html <<EOF
<!DOCTYPE html>
<html>
<head><title>My App</title></head>
<body><h1>Hello from Podman Desktop!</h1></body>
</html>
EOF

# Create the Containerfile
cat > Containerfile <<EOF
# Use Nginx as the base image
FROM docker.io/library/nginx:alpine

# Copy the web content
COPY index.html /usr/share/nginx/html/index.html

# Expose port 80
EXPOSE 80

# Nginx runs automatically as the default command
EOF
```

## Building an Image with the CLI

Build the image from the command line.

```bash
# Build the image with a tag
podman build -t my-web-app:v1.0 .

# Verify the image was created
podman images | grep my-web-app

# Test the image by running a container
podman run -d --name test-app -p 8080:80 my-web-app:v1.0
curl http://localhost:8080

# Clean up the test container
podman rm -f test-app
```

## Building an Image with Podman Desktop

In Podman Desktop, follow these steps:

1. Click on "Images" in the left sidebar
2. Click the "Build" button at the top
3. Select the Containerfile path (browse to your project directory)
4. Set the build context directory (usually the same directory as the Containerfile)
5. Enter the image name and tag (e.g., `my-web-app:v1.0`)
6. Click "Build" to start the build process

The build output is displayed in real time, showing each layer being created.

## Building with Build Arguments

Pass build-time variables to customize the image.

```bash
# Create a Containerfile with build arguments
cat > Containerfile.args <<EOF
# Accept build arguments
ARG BASE_IMAGE=docker.io/library/nginx:alpine
ARG APP_VERSION=1.0.0

# Use the base image argument
FROM \${BASE_IMAGE}

# Label with version information
LABEL version=\${APP_VERSION}
LABEL maintainer="team@example.com"

# Copy application files
COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
EOF

# Build with custom arguments
podman build \
    -f Containerfile.args \
    --build-arg APP_VERSION=2.0.0 \
    --build-arg BASE_IMAGE=docker.io/library/nginx:1.25-alpine \
    -t my-web-app:v2.0 .

# Verify the labels
podman inspect my-web-app:v2.0 | grep -A 2 "Labels"
```

## Multi-Stage Builds

Use multi-stage builds to create smaller production images.

```bash
# Create a Go application
cat > main.go <<EOF
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello from Go app built with Podman!")
    })
    http.ListenAndServe(":8080", nil)
}
EOF

cat > go.mod <<EOF
module myapp
go 1.21
EOF

# Create a multi-stage Containerfile
cat > Containerfile.multistage <<EOF
# Stage 1: Build the Go binary
FROM docker.io/library/golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod .
COPY main.go .

# Build a statically linked binary
RUN CGO_ENABLED=0 go build -o server main.go

# Stage 2: Create the minimal production image
FROM docker.io/library/alpine:3.19

# Copy only the binary from the builder stage
COPY --from=builder /app/server /usr/local/bin/server

EXPOSE 8080
CMD ["server"]
EOF

# Build the multi-stage image
podman build -f Containerfile.multistage -t my-go-app:v1.0 .

# Check the image size (should be very small)
podman images | grep my-go-app
```

## Building with a .containerignore File

Exclude files from the build context to speed up builds.

```bash
# Create a .containerignore file
cat > .containerignore <<EOF
# Ignore version control
.git
.gitignore

# Ignore documentation
*.md
docs/

# Ignore test files
*_test.go
tests/

# Ignore local configuration
.env
.env.local
EOF

# Build with the ignore file in place
podman build -t my-web-app:v1.1 .
```

## Tagging Built Images

Add additional tags to your built images.

```bash
# Tag an existing image with a new name
podman tag my-web-app:v1.0 registry.example.com/myorg/web-app:v1.0
podman tag my-web-app:v1.0 registry.example.com/myorg/web-app:latest

# Verify the tags
podman images | grep web-app
```

In Podman Desktop, you can also tag images from the Images list by clicking on an image and using the tag option.

## Viewing Build History

Inspect the layers and history of a built image.

```bash
# View the image build history
podman history my-web-app:v1.0

# View detailed layer information
podman history --no-trunc my-web-app:v1.0
```

In Podman Desktop, click on an image to see its details, including layers and history.

## Summary

Building images with Podman Desktop combines a visual interface with the full power of Podman builds. You can create images from Containerfiles using the GUI's build dialog or the `podman build` CLI command. Both approaches support build arguments, multi-stage builds, and `.containerignore` files for optimized builds. Podman Desktop shows build output in real time and lets you manage, tag, and inspect built images from the Images tab. This gives you a complete image building workflow within a single graphical application.
