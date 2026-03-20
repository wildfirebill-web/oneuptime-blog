# How to Implement a Chat Application with Python Sockets over IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, TCP, Chat, Sockets, IPv4, Threading, Networking

Description: Learn how to build a multi-client chat application in Python using IPv4 TCP sockets with threading for real-time message broadcasting.

## Architecture

The chat server maintains a list of connected clients and broadcasts every received message to all other clients. Each client runs two threads: one for receiving messages and one for sending user input.

## Chat Server

```python
import socket
import threading

HOST = "0.0.0.0"
PORT = 9006

# Shared state protected by a lock

clients: dict[socket.socket, str] = {}   # socket -> username
lock = threading.Lock()


def broadcast(message: bytes, sender: socket.socket = None) -> None:
    """Send a message to all connected clients except the sender."""
    with lock:
        for client_sock in list(clients.keys()):
            if client_sock is not sender:
                try:
                    client_sock.sendall(message)
                except OSError:
                    pass


def handle_client(conn: socket.socket, addr: tuple) -> None:
    """Handle a single client: register, relay messages, cleanup on disconnect."""
    # First message from client is their username
    try:
        username = conn.recv(64).decode("utf-8").strip() or "anonymous"
    except OSError:
        conn.close()
        return

    with lock:
        clients[conn] = username

    broadcast(f"[Server] {username} has joined the chat!\n".encode(), sender=conn)
    print(f"[+] {username} connected from {addr}")

    try:
        while True:
            data = conn.recv(1024)
            if not data:
                break   # Client disconnected

            msg = f"[{username}] {data.decode('utf-8', errors='replace')}"
            print(msg.strip())
            broadcast(msg.encode(), sender=conn)

    except (ConnectionResetError, BrokenPipeError):
        pass
    finally:
        with lock:
            clients.pop(conn, None)
        conn.close()
        broadcast(f"[Server] {username} has left the chat.\n".encode())
        print(f"[-] {username} disconnected")


def run_server():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
        srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        srv.bind((HOST, PORT))
        srv.listen(50)
        print(f"Chat server on {HOST}:{PORT}")

        while True:
            try:
                conn, addr = srv.accept()
                t = threading.Thread(target=handle_client, args=(conn, addr), daemon=True)
                t.start()
            except KeyboardInterrupt:
                print("Server stopped")
                break


if __name__ == "__main__":
    run_server()
```

## Chat Client

```python
import socket
import threading
import sys

SERVER_HOST = "127.0.0.1"
SERVER_PORT = 9006


def receive_messages(sock: socket.socket) -> None:
    """Background thread: print messages from the server."""
    while True:
        try:
            data = sock.recv(1024)
            if not data:
                print("\n[Disconnected from server]")
                sys.exit(0)
            print(data.decode("utf-8", errors="replace"), end="")
        except OSError:
            break


def run_client():
    username = input("Enter your username: ").strip()

    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as client:
        client.connect((SERVER_HOST, SERVER_PORT))

        # Send username as the first message
        client.sendall(username.encode("utf-8"))

        # Start receive thread
        recv_thread = threading.Thread(target=receive_messages, args=(client,), daemon=True)
        recv_thread.start()

        print(f"Connected as '{username}'. Type messages and press Enter.")

        try:
            while True:
                msg = input()
                if msg.lower() == "/quit":
                    break
                client.sendall(f"{msg}\n".encode("utf-8"))
        except (KeyboardInterrupt, EOFError):
            pass

    print("Goodbye!")


if __name__ == "__main__":
    run_client()
```

## Running the Chat

```bash
# Terminal 1: Start server
python3 chat_server.py

# Terminal 2: First client
python3 chat_client.py

# Terminal 3: Second client
python3 chat_client.py
```

## Conclusion

A multi-client chat server requires a shared client registry, thread-safe broadcasting with a lock, and per-client handler threads. The client uses a background receive thread so input and output can run simultaneously. This pattern is the foundation for any real-time messaging system over TCP.
