# How to Debug gRPC Connection Issues on IPv4 Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: gRPC, Debugging, IPv4, Networking, Python, Go

Description: Learn how to debug gRPC connection issues on IPv4 networks using grpcurl, verbose logging, tcpdump, and interceptors to identify and resolve common problems.

## Quick Checklist

```
1. Can you reach the server IP and port? (ping, telnet, nc)
2. Is the server actually listening on that port? (ss, netstat)
3. Is TLS configured correctly on both sides?
4. Are the proto definitions in sync?
5. Does grpcurl work? (tests layer by layer)
6. Are there firewall rules blocking the port?
7. Is the server returning an error status?
```

## grpcurl: The gRPC Equivalent of curl

```bash
# Install
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# List available services (requires server-side reflection)
grpcurl -plaintext 192.168.1.10:50051 list

# List methods of a service
grpcurl -plaintext 192.168.1.10:50051 list helloworld.Greeter

# Describe a message type
grpcurl -plaintext 192.168.1.10:50051 describe helloworld.HelloRequest

# Call a method
grpcurl -plaintext -d '{"name":"debug"}' \
    192.168.1.10:50051 helloworld.Greeter/SayHello

# With TLS
grpcurl -cacert ca.crt -d '{"name":"debug"}' \
    192.168.1.10:50051 helloworld.Greeter/SayHello

# Verbose output to see headers and trailers
grpcurl -plaintext -v -d '{"name":"debug"}' \
    192.168.1.10:50051 helloworld.Greeter/SayHello
```

## Python: Logging Interceptor

```python
import grpc
import logging

log = logging.getLogger("grpc.debug")

class DebugInterceptor(grpc.UnaryUnaryClientInterceptor):
    def intercept_unary_unary(self, continuation, client_call_details, request):
        log.info("→ %s | request=%s", client_call_details.method, request)
        response = continuation(client_call_details, request)
        try:
            result = response.result()
            log.info("← %s | response=%s", client_call_details.method, result)
            return response
        except grpc.RpcError as e:
            log.error("← %s | error=%s details=%s",
                      client_call_details.method, e.code(), e.details())
            raise

channel = grpc.insecure_channel("192.168.1.10:50051")
channel = grpc.intercept_channel(channel, DebugInterceptor())
```

## Go: Logging Interceptor

```go
import (
    "context"
    "log"
    "time"
    "google.golang.org/grpc"
)

func loggingInterceptor(
    ctx context.Context,
    method string,
    req, reply interface{},
    cc *grpc.ClientConn,
    invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption,
) error {
    start := time.Now()
    err := invoker(ctx, method, req, reply, cc, opts...)
    log.Printf("method=%s duration=%s err=%v", method, time.Since(start), err)
    return err
}

conn, _ := grpc.NewClient(
    "192.168.1.10:50051",
    grpc.WithUnaryInterceptor(loggingInterceptor),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

## Network-Level Debugging

```bash
# Capture gRPC traffic (HTTP/2 over port 50051)
tcpdump -i eth0 'tcp port 50051' -w grpc.pcap

# Open in Wireshark and look for:
# - TCP SYN/ACK (connection established)
# - TLS ClientHello/ServerHello (if TLS)
# - HTTP/2 SETTINGS frames
# - HTTP/2 HEADERS frames with :path = /service/Method
# - HTTP/2 DATA frames (request/response bodies)
# - GOAWAY frames (server closing connection)

# Check certificate expiry
openssl s_client -connect 192.168.1.10:50051 </dev/null 2>&1 | \
    openssl x509 -noout -dates
```

## Common Error Messages and Fixes

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| `UNAVAILABLE: Connection refused` | Server not running or wrong port | Start server, check port |
| `UNAVAILABLE: failed to connect` | Firewall, wrong IP | `ping`, `nc -zv ip port` |
| `DEADLINE_EXCEEDED` | Server slow or network issue | Increase timeout, profile server |
| `UNAUTHENTICATED` | TLS cert mismatch | Check CA bundle, cert expiry |
| `UNIMPLEMENTED` | Method not registered | Check proto version, re-register |
| `RESOURCE_EXHAUSTED` | Max connections exceeded | Increase server limits |

## Conclusion

`grpcurl` should be your first tool — it tests the entire gRPC stack including reflection, TLS, and method dispatch. Enable server-side reflection in development so you can introspect live services. Use interceptors for per-call logging without cluttering business logic. `tcpdump` + Wireshark reveals TLS and HTTP/2 framing issues invisible at the gRPC level. Most connectivity issues come down to the server binding to the wrong address, a firewall rule, or expired certificates.
