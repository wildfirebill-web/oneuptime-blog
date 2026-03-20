# How to Implement the Client-Server Pattern with IPv4 TCP in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, TCP, IPv4, Client-Server, Networking, ServerSocket

Description: Learn how to implement the client-server pattern with IPv4 TCP in Java using ServerSocket and Socket, with multi-threaded handling, DataInputStream/DataOutputStream framing, and graceful shutdown.

## Echo Server

```java
import java.io.*;
import java.net.*;

public class EchoServer {
    public static void main(String[] args) throws IOException {
        // Bind to all IPv4 interfaces on port 9000
        InetAddress bindAddr = InetAddress.getByName("0.0.0.0");
        try (ServerSocket server = new ServerSocket(9000, 50, bindAddr)) {
            System.out.println("Echo server on 0.0.0.0:9000");
            while (true) {
                Socket client = server.accept();
                new Thread(() -> handle(client)).start();
            }
        }
    }

    static void handle(Socket conn) {
        try (conn) {
            System.out.printf("[+] %s%n", conn.getRemoteSocketAddress());
            InputStream  in  = conn.getInputStream();
            OutputStream out = conn.getOutputStream();
            byte[] buf = new byte[4096];
            int n;
            while ((n = in.read(buf)) != -1) {
                out.write(buf, 0, n);
            }
        } catch (IOException e) {
            System.err.println("Handler error: " + e.getMessage());
        }
    }
}
```

## Echo Client

```java
import java.io.*;
import java.net.*;

public class EchoClient {
    public static void main(String[] args) throws IOException {
        try (Socket s = new Socket("192.168.1.10", 9000)) {
            s.setSoTimeout(5000);
            OutputStream out = s.getOutputStream();
            InputStream  in  = s.getInputStream();

            out.write("Hello, server!".getBytes());
            out.flush();

            byte[] buf = new byte[4096];
            int n = in.read(buf);
            System.out.printf("Received: %s%n", new String(buf, 0, n));
        }
    }
}
```

## Length-Prefixed Framing with DataStreams

```java
import java.io.*;
import java.net.*;

public class FramedServer {

    static void sendMsg(DataOutputStream out, byte[] payload) throws IOException {
        out.writeInt(payload.length);   // 4-byte big-endian length
        out.write(payload);
        out.flush();
    }

    static byte[] recvMsg(DataInputStream in) throws IOException {
        int length = in.readInt();      // blocking read of 4-byte header
        byte[] payload = new byte[length];
        in.readFully(payload);          // block until all bytes arrive
        return payload;
    }

    static void handle(Socket conn) {
        try (conn) {
            DataInputStream  din  = new DataInputStream(conn.getInputStream());
            DataOutputStream dout = new DataOutputStream(conn.getOutputStream());
            while (true) {
                byte[] msg = recvMsg(din);
                sendMsg(dout, msg);   // echo framed message back
            }
        } catch (EOFException ignored) {
            // client disconnected
        } catch (IOException e) {
            System.err.println(e.getMessage());
        }
    }

    public static void main(String[] args) throws IOException {
        ServerSocket server = new ServerSocket(9000);
        System.out.println("Framed echo server on :9000");
        while (true) {
            Socket client = server.accept();
            new Thread(() -> handle(client)).start();
        }
    }
}
```

## Thread Pool Server

```java
import java.io.*;
import java.net.*;
import java.util.concurrent.*;

public class PooledServer {
    public static void main(String[] args) throws IOException {
        ExecutorService pool = Executors.newFixedThreadPool(20);
        ServerSocket server  = new ServerSocket(9000);
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            pool.shutdown();
            try { server.close(); } catch (IOException ignored) {}
        }));

        System.out.println("Pooled server on :9000");
        while (!server.isClosed()) {
            try {
                final Socket client = server.accept();
                pool.submit(() -> EchoServer.handle(client));
            } catch (IOException e) {
                if (server.isClosed()) break;
            }
        }
    }
}
```

## Conclusion

Java's `ServerSocket(port, backlog, bindAddr)` provides fine-grained control over bind address and connection backlog. `DataInputStream.readFully()` is the reliable way to read an exact number of bytes - it retries internally until all bytes arrive. Use `ExecutorService.newFixedThreadPool` to limit concurrency and prevent thread exhaustion under load. Register a shutdown hook to close the `ServerSocket` (which unblocks `accept()`) and drain the thread pool gracefully. For Java 21+, consider virtual threads (`Executors.newVirtualThreadPerTaskExecutor()`) for massive concurrency without tuning pool sizes.
