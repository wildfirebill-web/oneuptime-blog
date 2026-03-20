# How to Configure gRPC Servers with IPv6 in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: gRPC, Java, IPv6, Spring Boot, API

Description: Configure Java gRPC servers using grpc-java to listen on IPv6 addresses and handle dual-stack deployments.

## Dependencies (Maven)

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>1.63.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>1.63.0</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.63.0</version>
    </dependency>
</dependencies>
```

## Step 1: gRPC Server on IPv6

```java
// HelloWorldServer.java
import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.netty.shaded.io.grpc.netty.NettyServerBuilder;

import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.util.concurrent.TimeUnit;

public class HelloWorldServer {
    private Server server;

    public void start() throws Exception {
        // Bind to all IPv6 interfaces using ::
        InetAddress ipv6WildCard = InetAddress.getByName("::");
        InetSocketAddress listenAddress = new InetSocketAddress(ipv6WildCard, 50051);

        server = NettyServerBuilder
            .forAddress(listenAddress)
            .addService(new GreeterImpl())
            .build()
            .start();

        System.out.println("gRPC server started on [::]:50051");

        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            try {
                server.shutdown().awaitTermination(30, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }));
    }

    public static void main(String[] args) throws Exception {
        HelloWorldServer server = new HelloWorldServer();
        server.start();
        server.server.awaitTermination();
    }
}
```

## Step 2: Service Implementation

```java
// GreeterImpl.java
import io.grpc.stub.StreamObserver;
import io.grpc.Context;

public class GreeterImpl extends GreeterGrpc.GreeterImplBase {

    @Override
    public void sayHello(HelloRequest request, StreamObserver<HelloReply> responseObserver) {
        // Get client's peer address (includes IPv6)
        String peerAddress = io.grpc.Grpc.TRANSPORT_ATTR_REMOTE_ADDR
            .get(io.grpc.Attributes.EMPTY);

        System.out.println("Request from: " + peerAddress);

        HelloReply reply = HelloReply.newBuilder()
            .setMessage("Hello, " + request.getName())
            .build();

        responseObserver.onNext(reply);
        responseObserver.onCompleted();
    }
}
```

## Step 3: gRPC Client Connecting to IPv6

```java
// HelloWorldClient.java
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.netty.shaded.io.grpc.netty.NettyChannelBuilder;

import java.net.InetAddress;
import java.net.InetSocketAddress;

public class HelloWorldClient {

    public static void main(String[] args) throws Exception {
        // Connect to IPv6 gRPC server
        InetAddress ipv6Server = InetAddress.getByName("2001:db8::1");
        InetSocketAddress serverAddress = new InetSocketAddress(ipv6Server, 50051);

        ManagedChannel channel = NettyChannelBuilder
            .forAddress(serverAddress)
            .usePlaintext()
            .build();

        GreeterGrpc.GreeterBlockingStub stub = GreeterGrpc.newBlockingStub(channel);

        HelloReply response = stub.sayHello(
            HelloRequest.newBuilder().setName("World").build()
        );

        System.out.println("Response: " + response.getMessage());
        channel.shutdown();
    }
}
```

## Step 4: Spring Boot gRPC with IPv6

Using `grpc-spring-boot-starter`:

```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
```

```yaml
# application.yml

grpc:
  server:
    # Bind to all IPv6 interfaces
    address: "[::]"
    port: 50051
  client:
    my-service:
      # Connect to IPv6 server
      address: "static://[2001:db8::1]:50051"
      negotiation-type: PLAINTEXT
```

```java
@GrpcService
public class GreeterService extends GreeterGrpc.GreeterImplBase {

    @Override
    public void sayHello(HelloRequest request, StreamObserver<HelloReply> response) {
        response.onNext(HelloReply.newBuilder()
            .setMessage("Hello " + request.getName())
            .build());
        response.onCompleted();
    }
}
```

## Step 5: Extract IPv6 Client Address

```java
import io.grpc.stub.ServerCallStreamObserver;
import io.grpc.Grpc;
import java.net.InetSocketAddress;

public class GreeterImpl extends GreeterGrpc.GreeterImplBase {

    @Override
    public void sayHello(HelloRequest request, StreamObserver<HelloReply> responseObserver) {
        // Get remote address from context
        io.grpc.ServerCall<?, ?> call = io.grpc.ServerInterceptors.CALL_SERVER_CALL.get();

        // Alternative: use interceptor to capture address
        // See GrpcClientAddressInterceptor below
        System.out.println("Processing request for: " + request.getName());
        responseObserver.onNext(HelloReply.newBuilder().setMessage("Hello!").build());
        responseObserver.onCompleted();
    }
}
```

## Testing

```bash
# Test with grpcurl
grpcurl -plaintext '[2001:db8::1]:50051' list
grpcurl -plaintext '[2001:db8::1]:50051' helloworld.Greeter/SayHello

# Health check
grpcurl -plaintext '[2001:db8::1]:50051' grpc.health.v1.Health/Check
```

## Monitoring with OneUptime

Use [OneUptime](https://oneuptime.com) to monitor your Java gRPC service over IPv6. Set up TCP availability monitors on port 50051 and health check monitors for the gRPC health protocol endpoint.

## Conclusion

Java gRPC servers bind to IPv6 using `InetAddress.getByName("::")` and `NettyServerBuilder.forAddress()`. Clients connect using `InetSocketAddress` with the IPv6 address. Spring Boot gRPC starters accept `[::]` notation in configuration. All Java gRPC features work transparently with IPv6.
