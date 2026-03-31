# How to Create a TCP Server in Java Using ServerSocket with IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, TCP, Socket, IPv4, ServerSocket, Networking

Description: Learn how to create a TCP server in Java using ServerSocket with IPv4 addressing to accept client connections and exchange data.

## Basic TCP Echo Server

```java
import java.io.*;
import java.net.*;

public class TcpEchoServer {
    private static final String HOST = "0.0.0.0";
    private static final int PORT = 9000;

    public static void main(String[] args) throws IOException {
        // InetAddress.getByName("0.0.0.0") listens on all IPv4 interfaces
        InetAddress bindAddr = InetAddress.getByName(HOST);

        // ServerSocket(port, backlog, bindAddr)
        // backlog=50 means up to 50 queued connections
        try (ServerSocket serverSocket = new ServerSocket(PORT, 50, bindAddr)) {
            serverSocket.setReuseAddress(true);
            System.out.printf("TCP server listening on %s:%d%n", HOST, PORT);

            while (true) {
                // accept() blocks until a client connects
                Socket clientSocket = serverSocket.accept();
                System.out.printf("Client connected: %s%n", clientSocket.getRemoteSocketAddress());

                // Handle client in a new thread
                new Thread(() -> handleClient(clientSocket)).start();
            }
        }
    }

    private static void handleClient(Socket socket) {
        String addr = socket.getRemoteSocketAddress().toString();
        try (
            socket;
            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
            PrintWriter out = new PrintWriter(socket.getOutputStream(), true)
        ) {
            String line;
            while ((line = in.readLine()) != null) {
                System.out.printf("[%s] Received: %s%n", addr, line);
                out.println(line);  // Echo back
            }
        } catch (SocketException e) {
            System.out.printf("[%s] Connection reset%n", addr);
        } catch (IOException e) {
            System.err.printf("[%s] Error: %s%n", addr, e.getMessage());
        }
        System.out.printf("[%s] Disconnected%n", addr);
    }
}
```

## Binding to a Specific IPv4 Interface

```java
import java.net.*;

// Bind to a specific interface (e.g., 192.168.1.50)
InetAddress specificAddr = InetAddress.getByName("192.168.1.50");
ServerSocket serverSocket = new ServerSocket(9000, 50, specificAddr);
```

## Reading Binary Data

```java
import java.io.*;
import java.net.*;

private static void handleBinaryClient(Socket socket) throws IOException {
    try (
        socket;
        DataInputStream in = new DataInputStream(socket.getInputStream());
        DataOutputStream out = new DataOutputStream(socket.getOutputStream())
    ) {
        while (true) {
            // Read 4-byte message length header
            int msgLen = in.readInt();
            if (msgLen <= 0 || msgLen > 1024 * 1024) break; // Sanity check

            // Read exactly msgLen bytes
            byte[] data = new byte[msgLen];
            in.readFully(data);  // readFully ensures all bytes are read

            String message = new String(data, "UTF-8");
            System.out.println("Received: " + message);

            // Echo back
            byte[] response = ("Echo: " + message).getBytes("UTF-8");
            out.writeInt(response.length);
            out.write(response);
            out.flush();
        }
    }
}
```

## Setting Socket Options

```java
Socket clientSocket = serverSocket.accept();

// Set read timeout: throw SocketTimeoutException after 30 seconds of inactivity
clientSocket.setSoTimeout(30_000);

// Disable Nagle's algorithm for low-latency apps
clientSocket.setTcpNoDelay(true);

// Enable TCP keepalive
clientSocket.setKeepAlive(true);

// Set receive and send buffer sizes
clientSocket.setReceiveBufferSize(65536);
clientSocket.setSendBufferSize(65536);
```

## Graceful Server Shutdown

```java
import java.net.*;
import java.util.concurrent.atomic.*;

public class GracefulServer {
    private ServerSocket serverSocket;
    private final AtomicBoolean running = new AtomicBoolean(true);

    public void start(int port) throws IOException {
        serverSocket = new ServerSocket(port);
        System.out.println("Server started on port " + port);

        Runtime.getRuntime().addShutdownHook(new Thread(this::shutdown));

        while (running.get()) {
            try {
                Socket client = serverSocket.accept();
                new Thread(() -> handleClient(client)).start();
            } catch (SocketException e) {
                if (!running.get()) break; // Expected during shutdown
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    public void shutdown() {
        running.set(false);
        try {
            serverSocket.close();
        } catch (IOException e) { /* ignore */ }
        System.out.println("Server stopped");
    }

    private void handleClient(Socket socket) {
        // Handle client...
    }
}
```

## Conclusion

Java's `ServerSocket` binds to an IPv4 address and port, while `Socket` represents each client connection. Use `readFully()` for binary protocols to ensure complete reads, `setSoTimeout()` for idle connection management, and a shutdown hook for graceful termination. For high-concurrency servers, use `ExecutorService` with a thread pool instead of creating unbounded threads.
