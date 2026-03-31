# How to Build a Redis Client from Scratch Using RESP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RESP, Protocol, Client

Description: Build a minimal Redis client from scratch using raw TCP sockets and the RESP protocol, covering encoding commands and decoding responses.

---

Building a Redis client from scratch is the best way to understand how RESP works. This walkthrough creates a minimal, functional client in Python using only the standard library.

## Setting Up the TCP Connection

Redis listens on TCP port 6379 by default. A client is just a TCP socket that speaks RESP:

```python
import socket

class RedisClient:
    def __init__(self, host="localhost", port=6379):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.connect((host, port))
        self.file = self.sock.makefile("rb")  # buffered reader

    def close(self):
        self.sock.close()
```

## Encoding Commands as RESP Arrays

Every Redis command is sent as a RESP array of bulk strings:

```python
def encode_command(self, *args):
    parts = [f"*{len(args)}\r\n"]
    for arg in args:
        arg_bytes = str(arg).encode()
        parts.append(f"${len(arg_bytes)}\r\n")
        parts.append(arg_bytes.decode())
        parts.append("\r\n")
    return "".join(parts).encode()
```

Example - encoding `SET foo bar`:

```text
*3\r\n
$3\r\n
SET\r\n
$3\r\n
foo\r\n
$3\r\n
bar\r\n
```

## Parsing RESP Responses

The parser reads the type byte and delegates:

```python
def read_response(self):
    line = self.file.readline().decode().strip()
    prefix = line[0]
    data = line[1:]

    if prefix == "+":   # Simple string
        return data
    elif prefix == "-": # Error
        raise Exception(data)
    elif prefix == ":": # Integer
        return int(data)
    elif prefix == "$": # Bulk string
        length = int(data)
        if length == -1:
            return None
        value = self.file.read(length)
        self.file.read(2)  # consume \r\n
        return value.decode()
    elif prefix == "*": # Array
        count = int(data)
        if count == -1:
            return None
        return [self.read_response() for _ in range(count)]
    else:
        raise Exception(f"Unknown RESP type: {prefix}")
```

## Sending Commands

```python
def execute(self, *args):
    self.sock.sendall(self.encode_command(*args))
    return self.read_response()
```

## Testing the Client

```python
client = RedisClient()

# SET
result = client.execute("SET", "greeting", "hello")
print(result)  # OK

# GET
result = client.execute("GET", "greeting")
print(result)  # hello

# INCR
client.execute("SET", "counter", "0")
result = client.execute("INCR", "counter")
print(result)  # 1 (integer)

# DEL
result = client.execute("DEL", "greeting")
print(result)  # 1

# GET missing key
result = client.execute("GET", "missing")
print(result)  # None (nil)

client.close()
```

## Handling HGETALL

`HGETALL` returns a flat array in RESP2, so the client must convert pairs to a dict:

```python
def hgetall(self, key):
    raw = self.execute("HGETALL", key)
    return dict(zip(raw[::2], raw[1::2]))

client.execute("HSET", "user:1", "name", "Alice", "age", "30")
print(client.hgetall("user:1"))
# {"name": "Alice", "age": "30"}
```

## Adding Pipeline Support

Pipelines batch multiple commands and read all responses at once:

```python
def pipeline(self):
    return Pipeline(self)

class Pipeline:
    def __init__(self, client):
        self.client = client
        self.commands = []

    def execute(self, *args):
        self.commands.append(args)
        return self

    def run(self):
        payload = b"".join(
            self.client.encode_command(*cmd) for cmd in self.commands
        )
        self.client.sock.sendall(payload)
        return [self.client.read_response() for _ in self.commands]

pipe = client.pipeline()
pipe.execute("SET", "a", "1")
pipe.execute("SET", "b", "2")
pipe.execute("GET", "a")
results = pipe.run()
print(results)  # ["OK", "OK", "1"]
```

## Summary

A Redis client is a TCP socket that encodes commands as RESP arrays and decodes responses by reading the type prefix byte. The core logic is under 60 lines of Python. Building this from scratch makes RESP protocol handling intuitive and provides the foundation for understanding pipelining, transactions, and the RESP3 extensions.
