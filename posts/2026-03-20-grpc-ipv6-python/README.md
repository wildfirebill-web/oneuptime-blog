# How to Configure gRPC Servers with IPv6 in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: gRPC, Python, IPv6, API, Networking

Description: Configure Python gRPC servers and clients to use IPv6 addresses, with examples for both asyncio and synchronous implementations.

## Installation

```bash
# Install gRPC for Python
pip install grpcio grpcio-tools

# Verify installation
python -c "import grpc; print(grpc.__version__)"
```

## Step 1: Define a Proto File

```protobuf
// hello.proto
syntax = "proto3";

package helloworld;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

```bash
# Generate Python gRPC code
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. hello.proto
```

## Step 2: gRPC Server on IPv6

```python
# server.py
import grpc
from concurrent import futures
import hello_pb2
import hello_pb2_grpc

class GreeterServicer(hello_pb2_grpc.GreeterServicer):
    def SayHello(self, request, context):
        # Get client's IPv6 address from context
        peer = context.peer()
        print(f"Request from: {peer}")
        return hello_pb2.HelloReply(
            message=f"Hello, {request.name}! (your IP: {peer})"
        )

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    hello_pb2_grpc.add_GreeterServicer_to_server(GreeterServicer(), server)

    # Bind to all IPv6 interfaces — [::] is the IPv6 wildcard
    # Python gRPC uses [::]:port format
    listen_addr = "[::]:50051"
    server.add_insecure_port(listen_addr)

    server.start()
    print(f"gRPC server started on {listen_addr}")
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
```

## Step 3: gRPC Server with TLS on IPv6

```python
import grpc

def serve_with_tls():
    # Load TLS credentials
    with open("server.key", "rb") as f:
        private_key = f.read()
    with open("server.crt", "rb") as f:
        certificate_chain = f.read()

    server_credentials = grpc.ssl_server_credentials(
        [(private_key, certificate_chain)]
    )

    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    hello_pb2_grpc.add_GreeterServicer_to_server(GreeterServicer(), server)

    # Add TLS port on IPv6
    server.add_secure_port("[::]:443", server_credentials)

    server.start()
    print("Secure gRPC server on [::]:443")
    server.wait_for_termination()
```

## Step 4: gRPC Client Connecting to IPv6

```python
# client.py
import grpc
import hello_pb2
import hello_pb2_grpc

def run():
    # Connect to IPv6 gRPC server using [addr]:port format
    ipv6_target = "[2001:db8::1]:50051"

    with grpc.insecure_channel(ipv6_target) as channel:
        stub = hello_pb2_grpc.GreeterStub(channel)
        response = stub.SayHello(
            hello_pb2.HelloRequest(name="World"),
            timeout=5.0
        )
        print(f"Response: {response.message}")

if __name__ == "__main__":
    run()
```

## Step 5: Asyncio gRPC Server on IPv6

```python
# async_server.py
import asyncio
import grpc
import hello_pb2
import hello_pb2_grpc

class AsyncGreeterServicer(hello_pb2_grpc.GreeterServicer):
    async def SayHello(self, request, context):
        peer = context.peer()
        return hello_pb2.HelloReply(message=f"Hello async, {request.name}!")

async def serve():
    server = grpc.aio.server()
    hello_pb2_grpc.add_GreeterServicer_to_server(AsyncGreeterServicer(), server)

    # Listen on all IPv6 interfaces
    server.add_insecure_port("[::]:50051")

    await server.start()
    print("Async gRPC server on [::]:50051")
    await server.wait_for_termination()

if __name__ == "__main__":
    asyncio.run(serve())
```

## Step 6: Extract Client IPv6 Address in Interceptor

```python
class IPv6LoggingInterceptor(grpc.ServerInterceptor):
    def intercept_service(self, continuation, handler_call_details):
        # Log the peer IPv6 address
        peer = handler_call_details.invocation_metadata
        print(f"Incoming connection metadata: {peer}")
        return continuation(handler_call_details)

# For per-call context:
class GreeterServicer(hello_pb2_grpc.GreeterServicer):
    def SayHello(self, request, context):
        # context.peer() returns "ipv6:[2001:db8::1]:12345"
        peer_address = context.peer()
        # Parse out the IPv6 address
        if peer_address.startswith("ipv6:"):
            # Format: ipv6:[addr]:port
            addr_part = peer_address[5:]  # Remove "ipv6:"
            ip = addr_part.rsplit(":", 1)[0].strip("[]")
            print(f"Client IPv6: {ip}")
        return hello_pb2.HelloReply(message="Hello!")
```

## Testing

```bash
# Test with grpcurl
grpcurl -plaintext '[2001:db8::1]:50051' helloworld.Greeter/SayHello

# Or with Python test client
python client.py
```

## Monitoring with OneUptime

Use [OneUptime](https://oneuptime.com) to monitor your Python gRPC service availability over IPv6. Configure TCP monitors on port 50051 at the IPv6 address and set up health check monitors using the gRPC health protocol.

## Conclusion

Python gRPC servers bind to IPv6 using `[::]:port` as the address string. Clients connect using `[ipv6addr]:port`. Access client IPv6 addresses via `context.peer()` which returns the address in `ipv6:[addr]:port` format. All Python gRPC features work seamlessly with IPv6.
