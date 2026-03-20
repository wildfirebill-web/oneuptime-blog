# How to Bind a REST API Server to a Specific IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: REST API, IPv4, Networking, Python, Node.js, Java

Description: Learn how to bind REST API servers to specific IPv4 addresses in Python, Node.js, Java, and Go to control network exposure, separate internal and public interfaces, and improve security.

## Why Bind to a Specific Address?

Binding to `0.0.0.0` accepts connections on all interfaces. Binding to a specific IP restricts the server to one interface - useful to keep an admin API private (`127.0.0.1`) while exposing a public API on the machine's public IP.

```text
Bind 0.0.0.0:8080  → accepts on all interfaces (eth0, lo, vpn0, ...)
Bind 127.0.0.1:8080 → localhost only
Bind 192.168.1.10:8080 → one specific interface
```

## Python: Flask

```python
from flask import Flask

app = Flask(__name__)

@app.route("/health")
def health():
    return {"status": "ok"}

if __name__ == "__main__":
    # Bind only to internal network interface
    app.run(host="192.168.1.10", port=8080, debug=False)
    # Use host="0.0.0.0" to accept on all interfaces
```

## Python: FastAPI with uvicorn

```python
import uvicorn
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
async def health():
    return {"status": "ok"}

if __name__ == "__main__":
    uvicorn.run(app, host="192.168.1.10", port=8080)
```

## Node.js: Express

```javascript
const express = require("express");
const app = express();

app.get("/health", (req, res) => {
    res.json({ status: "ok" });
});

// Bind to specific IPv4 address
const HOST = "192.168.1.10";
const PORT = 8080;

app.listen(PORT, HOST, () => {
    console.log(`Server listening on http://${HOST}:${PORT}`);
});
```

## Go: net/http

```go
package main

import (
    "fmt"
    "net/http"
)

func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    fmt.Fprintln(w, `{"status":"ok"}`)
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/health", healthHandler)

    // Bind to specific IPv4 interface
    addr := "192.168.1.10:8080"
    fmt.Printf("Listening on %s\n", addr)
    if err := http.ListenAndServe(addr, mux); err != nil {
        panic(err)
    }
}
```

## Java: Spring Boot

```properties
# application.properties

server.address=192.168.1.10
server.port=8080
```

```java
// Or programmatically
@SpringBootApplication
public class ApiApplication {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(ApiApplication.class);
        app.setDefaultProperties(Map.of(
            "server.address", "192.168.1.10",
            "server.port",    "8080"
        ));
        app.run(args);
    }
}
```

## Conclusion

All major web frameworks accept a host/address parameter alongside the port. Use `127.0.0.1` for loopback-only (admin APIs, metrics endpoints), the machine's LAN IP for internal-only exposure, and `0.0.0.0` when the service should be reachable on any interface. Document the chosen bind address in deployment configuration alongside firewall rules so the intent is explicit and auditable.
