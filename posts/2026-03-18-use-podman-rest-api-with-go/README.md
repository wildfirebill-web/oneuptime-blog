# How to Use the Podman REST API with Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, REST API, Go, Golang, Containers

Description: Learn how to interact with the Podman REST API using Go, including building HTTP clients for Unix sockets, managing containers, and creating production-ready tooling.

---

> Go is a natural fit for container management tooling. Its strong standard library, excellent HTTP client support, and efficient concurrency model make it ideal for building tools that interact with the Podman REST API. In fact, Podman itself is written in Go.

Go's compiled nature, small binary sizes, and built-in concurrency primitives make it the language of choice for infrastructure tooling. The standard library includes everything you need to communicate with the Podman REST API over Unix sockets, with no external dependencies required for basic operations.

This guide covers how to build a Go client for the Podman REST API, from basic HTTP requests over Unix sockets to a complete container management library.

---

## Setting Up the Project

Initialize a new Go module:

```bash
mkdir podman-client && cd podman-client
go mod init podman-client
```

## Connecting to the Podman Socket

Go's `net/http` package supports custom transports, which allows you to connect through Unix sockets:

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net"
    "net/http"
    "os"
    "time"
)

func newPodmanClient(socketPath string) *http.Client {
    return &http.Client{
        Transport: &http.Transport{
            DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
                dialer := net.Dialer{}
                return dialer.DialContext(ctx, "unix", socketPath)
            },
        },
        Timeout: 30 * time.Second,
    }
}

func main() {
    client := newPodmanClient("/run/podman/podman.sock")

    resp, err := client.Get("http://localhost/v4.0.0/libpod/info")
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
    defer resp.Body.Close()

    var info map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&info)

    host := info["host"].(map[string]interface{})
    fmt.Printf("Connected to: %s\n", host["hostname"])
}
```

## Building a Podman Client Library

Create a structured client with types and methods:

```go
package main

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net"
    "net/http"
    "time"
)

const apiVersion = "v4.0.0"

// PodmanClient wraps HTTP client for Podman API
type PodmanClient struct {
    httpClient *http.Client
    baseURL    string
}

// Container represents a container listing entry
type Container struct {
    ID    string   `json:"Id"`
    Names []string `json:"Names"`
    State string   `json:"State"`
    Image string   `json:"Image"`
}

// ContainerSpec defines the configuration for creating a container
type ContainerSpec struct {
    Image        string            `json:"image"`
    Name         string            `json:"name,omitempty"`
    Command      []string          `json:"command,omitempty"`
    Env          map[string]string `json:"env,omitempty"`
    PortMappings []PortMapping     `json:"portmappings,omitempty"`
}

// PortMapping defines a port mapping
type PortMapping struct {
    ContainerPort int    `json:"container_port"`
    HostPort      int    `json:"host_port"`
    Protocol      string `json:"protocol"`
}

// CreateResponse is returned when a container is created
type CreateResponse struct {
    ID       string   `json:"Id"`
    Warnings []string `json:"Warnings"`
}

// ContainerInspect holds detailed container info
type ContainerInspect struct {
    ID    string `json:"Id"`
    Name  string `json:"Name"`
    State struct {
        Status  string `json:"Status"`
        Running bool   `json:"Running"`
        Pid     int    `json:"Pid"`
    } `json:"State"`
}

// NewPodmanClient creates a new client
func NewPodmanClient(socketPath string) *PodmanClient {
    return &PodmanClient{
        httpClient: &http.Client{
            Transport: &http.Transport{
                DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
                    dialer := net.Dialer{}
                    return dialer.DialContext(ctx, "unix", socketPath)
                },
            },
            Timeout: 30 * time.Second,
        },
        baseURL: fmt.Sprintf("http://localhost/%s/libpod", apiVersion),
    }
}

func (c *PodmanClient) doRequest(method, endpoint string, body interface{}) (*http.Response, error) {
    var bodyReader io.Reader
    if body != nil {
        jsonData, err := json.Marshal(body)
        if err != nil {
            return nil, fmt.Errorf("marshal body: %w", err)
        }
        bodyReader = bytes.NewReader(jsonData)
    }

    req, err := http.NewRequest(method, c.baseURL+endpoint, bodyReader)
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }

    if body != nil {
        req.Header.Set("Content-Type", "application/json")
    }

    return c.httpClient.Do(req)
}

// ListContainers returns all containers
func (c *PodmanClient) ListContainers(all bool) ([]Container, error) {
    endpoint := fmt.Sprintf("/containers/json?all=%t", all)
    resp, err := c.doRequest("GET", endpoint, nil)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var containers []Container
    if err := json.NewDecoder(resp.Body).Decode(&containers); err != nil {
        return nil, fmt.Errorf("decode response: %w", err)
    }
    return containers, nil
}

// CreateContainer creates a new container
func (c *PodmanClient) CreateContainer(spec ContainerSpec) (*CreateResponse, error) {
    resp, err := c.doRequest("POST", "/containers/create", spec)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusCreated {
        body, _ := io.ReadAll(resp.Body)
        return nil, fmt.Errorf("create failed (HTTP %d): %s", resp.StatusCode, body)
    }

    var result CreateResponse
    json.NewDecoder(resp.Body).Decode(&result)
    return &result, nil
}

// StartContainer starts a container by name or ID
func (c *PodmanClient) StartContainer(name string) error {
    resp, err := c.doRequest("POST", fmt.Sprintf("/containers/%s/start", name), nil)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusNoContent && resp.StatusCode != http.StatusOK {
        body, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("start failed (HTTP %d): %s", resp.StatusCode, body)
    }
    return nil
}

// StopContainer stops a container
func (c *PodmanClient) StopContainer(name string, timeout int) error {
    endpoint := fmt.Sprintf("/containers/%s/stop?timeout=%d", name, timeout)
    resp, err := c.doRequest("POST", endpoint, nil)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusNoContent && resp.StatusCode != http.StatusOK {
        body, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("stop failed (HTTP %d): %s", resp.StatusCode, body)
    }
    return nil
}

// RemoveContainer removes a container
func (c *PodmanClient) RemoveContainer(name string, force bool) error {
    endpoint := fmt.Sprintf("/containers/%s?force=%t", name, force)
    resp, err := c.doRequest("DELETE", endpoint, nil)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK && resp.StatusCode != http.StatusNoContent {
        body, _ := io.ReadAll(resp.Body)
        return fmt.Errorf("remove failed (HTTP %d): %s", resp.StatusCode, body)
    }
    return nil
}

// InspectContainer returns detailed container info
func (c *PodmanClient) InspectContainer(name string) (*ContainerInspect, error) {
    resp, err := c.doRequest("GET", fmt.Sprintf("/containers/%s/json", name), nil)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var inspect ContainerInspect
    if err := json.NewDecoder(resp.Body).Decode(&inspect); err != nil {
        return nil, fmt.Errorf("decode inspect: %w", err)
    }
    return &inspect, nil
}

// GetLogs retrieves container logs
func (c *PodmanClient) GetLogs(name string, tail int) (string, error) {
    endpoint := fmt.Sprintf("/containers/%s/logs?stdout=true&stderr=true&tail=%d", name, tail)
    resp, err := c.doRequest("GET", endpoint, nil)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return "", fmt.Errorf("read logs: %w", err)
    }
    return string(body), nil
}
```

## Using the Client

Here is a complete example that uses the client library:

```go
func main() {
    client := NewPodmanClient("/run/podman/podman.sock")

    // List running containers
    containers, err := client.ListContainers(false)
    if err != nil {
        fmt.Printf("Error listing containers: %v\n", err)
        return
    }

    fmt.Println("Running containers:")
    for _, c := range containers {
        fmt.Printf("  %s (%s) - %s\n", c.Names[0], c.ID[:12], c.State)
    }

    // Create and start a container
    spec := ContainerSpec{
        Image: "docker.io/library/nginx:alpine",
        Name:  "go-demo",
        PortMappings: []PortMapping{
            {ContainerPort: 80, HostPort: 9090, Protocol: "tcp"},
        },
    }

    result, err := client.CreateContainer(spec)
    if err != nil {
        fmt.Printf("Error creating container: %v\n", err)
        return
    }
    fmt.Printf("Created container: %s\n", result.ID[:12])

    if err := client.StartContainer("go-demo"); err != nil {
        fmt.Printf("Error starting container: %v\n", err)
        return
    }
    fmt.Println("Container started")

    // Inspect
    inspect, err := client.InspectContainer("go-demo")
    if err != nil {
        fmt.Printf("Error inspecting: %v\n", err)
        return
    }
    fmt.Printf("Status: %s, PID: %d\n", inspect.State.Status, inspect.State.Pid)

    // Clean up
    client.StopContainer("go-demo", 5)
    client.RemoveContainer("go-demo", false)
    fmt.Println("Container removed")
}
```

## Concurrent Container Operations

Go's goroutines make it easy to perform operations on multiple containers concurrently:

```go
func stopAllContainers(client *PodmanClient) error {
    containers, err := client.ListContainers(false)
    if err != nil {
        return err
    }

    errChan := make(chan error, len(containers))
    for _, container := range containers {
        go func(name string) {
            fmt.Printf("Stopping %s...\n", name)
            errChan <- client.StopContainer(name, 10)
        }(container.Names[0])
    }

    var errors []error
    for range containers {
        if err := <-errChan; err != nil {
            errors = append(errors, err)
        }
    }

    if len(errors) > 0 {
        return fmt.Errorf("failed to stop %d containers", len(errors))
    }
    return nil
}
```

## Streaming Logs

Stream container logs using Go's streaming HTTP support:

```go
func streamLogs(client *PodmanClient, containerName string) error {
    resp, err := client.doRequest("GET",
        fmt.Sprintf("/containers/%s/logs?follow=true&stdout=true&stderr=true", containerName),
        nil)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    buf := make([]byte, 4096)
    for {
        n, err := resp.Body.Read(buf)
        if n > 0 {
            fmt.Print(string(buf[:n]))
        }
        if err == io.EOF {
            break
        }
        if err != nil {
            return fmt.Errorf("read log stream: %w", err)
        }
    }
    return nil
}
```

## Health Check Endpoint

Build a simple health checker that monitors container states:

```go
func healthCheck(client *PodmanClient, containerNames []string) map[string]string {
    results := make(map[string]string)

    for _, name := range containerNames {
        inspect, err := client.InspectContainer(name)
        if err != nil {
            results[name] = "error: " + err.Error()
            continue
        }

        if inspect.State.Running {
            results[name] = "healthy"
        } else {
            results[name] = "down (" + inspect.State.Status + ")"
        }
    }

    return results
}

// Usage
func main() {
    client := NewPodmanClient("/run/podman/podman.sock")

    monitored := []string{"web-app", "database", "cache"}
    status := healthCheck(client, monitored)

    for name, state := range status {
        fmt.Printf("%-20s %s\n", name, state)
    }
}
```

## Error Handling Patterns

Implement robust error handling with custom error types:

```go
type PodmanError struct {
    StatusCode int
    Message    string
    Cause      string
}

func (e *PodmanError) Error() string {
    return fmt.Sprintf("podman API error (HTTP %d): %s", e.StatusCode, e.Message)
}

func (e *PodmanError) IsNotFound() bool {
    return e.StatusCode == http.StatusNotFound
}

func (e *PodmanError) IsConflict() bool {
    return e.StatusCode == http.StatusConflict
}

func checkResponse(resp *http.Response) error {
    if resp.StatusCode >= 200 && resp.StatusCode < 300 {
        return nil
    }

    var apiErr struct {
        Message string `json:"message"`
        Cause   string `json:"cause"`
    }
    json.NewDecoder(resp.Body).Decode(&apiErr)

    return &PodmanError{
        StatusCode: resp.StatusCode,
        Message:    apiErr.Message,
        Cause:      apiErr.Cause,
    }
}
```

## Conclusion

Go provides an excellent foundation for building Podman REST API clients. The standard library's HTTP and Unix socket support means you can build fully functional container management tools without any external dependencies. Go's type system helps ensure correct API interactions through structured request and response types, while goroutines enable efficient concurrent operations across multiple containers. Whether you are building a simple CLI tool or a complex orchestration platform, Go and the Podman API make a powerful combination.
