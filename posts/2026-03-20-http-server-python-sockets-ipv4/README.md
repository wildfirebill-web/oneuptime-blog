# How to Build a Simple HTTP Server Using Python Sockets and IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, HTTP, Sockets, IPv4, Web Server, Networking

Description: Learn how to build a minimal HTTP/1.1 server from scratch using Python raw sockets to understand the HTTP protocol at the byte level.

## HTTP at the Socket Level

HTTP is a text-based protocol over TCP. A request looks like:

```text
GET /path HTTP/1.1\r\n
Host: example.com\r\n
\r\n
```

A response looks like:

```text
HTTP/1.1 200 OK\r\n
Content-Type: text/plain\r\n
Content-Length: 13\r\n
\r\n
Hello, World!
```

## Minimal HTTP Server

```python
import socket
import threading

HOST = "0.0.0.0"
PORT = 8080


def parse_request(raw: bytes) -> tuple[str, str, dict]:
    """Parse an HTTP request into (method, path, headers)."""
    try:
        header_section, *_ = raw.split(b"\r\n\r\n", 1)
        lines = header_section.decode("utf-8").splitlines()
        method, path, _ = lines[0].split(" ", 2)
        headers = {}
        for line in lines[1:]:
            if ":" in line:
                key, value = line.split(":", 1)
                headers[key.strip().lower()] = value.strip()
        return method, path, headers
    except Exception:
        return "GET", "/", {}


def build_response(status: int, body: str, content_type: str = "text/plain") -> bytes:
    """Build a minimal HTTP/1.1 response."""
    status_text = {200: "OK", 404: "Not Found", 405: "Method Not Allowed"}.get(status, "Unknown")
    body_bytes = body.encode("utf-8")
    headers = (
        f"HTTP/1.1 {status} {status_text}\r\n"
        f"Content-Type: {content_type}; charset=utf-8\r\n"
        f"Content-Length: {len(body_bytes)}\r\n"
        f"Connection: close\r\n"
        f"\r\n"
    )
    return headers.encode("utf-8") + body_bytes


def handle_client(conn: socket.socket, addr: tuple) -> None:
    with conn:
        # Read request (simplified: assumes it fits in one recv)
        raw = conn.recv(8192)
        if not raw:
            return

        method, path, headers = parse_request(raw)
        print(f"{addr} {method} {path}")

        # Route requests
        if path == "/" and method == "GET":
            response = build_response(200, "Hello from Python socket HTTP server!\n")
        elif path == "/health" and method == "GET":
            response = build_response(200, '{"status":"ok"}', "application/json")
        elif method not in ("GET", "POST"):
            response = build_response(405, "Method Not Allowed")
        else:
            response = build_response(404, f"Not Found: {path}")

        conn.sendall(response)


def run_server():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
        srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        srv.bind((HOST, PORT))
        srv.listen(50)
        print(f"HTTP server on http://{HOST}:{PORT}")

        while True:
            try:
                conn, addr = srv.accept()
                threading.Thread(target=handle_client, args=(conn, addr), daemon=True).start()
            except KeyboardInterrupt:
                print("Server stopped")
                break


if __name__ == "__main__":
    run_server()
```

## Testing the Server

```bash
# Test root endpoint

curl http://localhost:8080/

# Test health endpoint
curl http://localhost:8080/health

# Test 404
curl http://localhost:8080/missing
```

## Reading POST Body

```python
def parse_post_body(raw: bytes) -> str:
    """Extract the body from a POST request."""
    parts = raw.split(b"\r\n\r\n", 1)
    if len(parts) > 1:
        return parts[1].decode("utf-8", errors="replace")
    return ""
```

## A Note on Production Use

This raw socket HTTP server is for learning purposes. For production, use a proper WSGI/ASGI server like `gunicorn`, `uvicorn`, or Python's built-in `http.server`. Real HTTP parsing handles chunked transfer encoding, keep-alive, pipelining, and many edge cases not covered here.

## Conclusion

Building an HTTP server on raw sockets reveals exactly how the protocol works: parse a text request, route on method and path, format a text response with headers. The `\r\n\r\n` boundary separates headers from body. This understanding is invaluable when debugging proxies, load balancers, and framework behavior.
