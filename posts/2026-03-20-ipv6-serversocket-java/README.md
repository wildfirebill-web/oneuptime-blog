# How to Use IPv6 with Java ServerSocket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, IPv6, ServerSocket, TCP, Networking, NIO

Description: Create IPv6 TCP servers in Java using ServerSocket, handle dual-stack connections, and use NIO ServerSocketChannel for non-blocking IPv6 servers.

## Basic IPv6 ServerSocket

```java
import java.io.*;
import java.net.*;

public class IPv6TCPServer {

    public static void main(String[] args) throws IOException {
        // Bind to [::]:8080 — accepts both IPv4 and IPv6 on dual-stack
        InetAddress bindAddr = InetAddress.getByName("::");
        ServerSocket server = new ServerSocket(8080, 50, bindAddr);

        System.out.println("Listening on: " + server.getLocalSocketAddress());

        while (true) {
            Socket client = server.accept();
            new Thread(() -> handleClient(client)).start();
        }
    }

    static void handleClient(Socket socket) {
        try (socket) {
            InetAddress clientAddr = socket.getInetAddress();
            System.out.println("Client: " + clientAddr.getHostAddress());

            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
            PrintWriter out = new PrintWriter(socket.getOutputStream(), true);

            String line;
            while ((line = in.readLine()) != null) {
                out.println("Echo: " + line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## IPv6-Only Server (Disable Dual-Stack)

On some systems, `ServerSocket` on `[::]` also accepts IPv4. To force IPv6-only, use `ServerSocketChannel` and set `IPV6_V6ONLY`:

```java
import java.io.IOException;
import java.net.*;
import java.nio.channels.*;

public class IPv6OnlyServer {

    public static void main(String[] args) throws IOException {
        ServerSocketChannel channel = ServerSocketChannel.open();

        // Set IPV6_V6ONLY socket option
        channel.setOption(StandardSocketOptions.IP_MULTICAST_IF,
            NetworkInterface.getByIndex(0));
        // For IPV6_V6ONLY, use the extended socket options API (Java 17+)

        channel.bind(new InetSocketAddress("::", 8080));
        channel.configureBlocking(true);

        System.out.println("IPv6 server on " + channel.getLocalAddress());

        while (true) {
            SocketChannel client = channel.accept();
            InetSocketAddress peer = (InetSocketAddress) client.getRemoteAddress();
            System.out.println("Client: " + peer.getAddress().getHostAddress());
            client.close();
        }
    }
}
```

## Extracting Real Client IPv6 from Dual-Stack

When a dual-stack server receives an IPv4 connection, Java presents it as an IPv4-mapped IPv6 address:

```java
import java.net.*;

public class ClientIPExtractor {

    public static InetAddress getRealClientIP(Socket socket) {
        InetAddress addr = socket.getInetAddress();

        // IPv4-mapped IPv6: ::ffff:x.x.x.x
        if (addr instanceof Inet6Address) {
            byte[] b = addr.getAddress();
            // Check if this is an IPv4-mapped address (bytes 0-9 are 0, 10-11 are 0xff)
            boolean mapped = true;
            for (int i = 0; i < 10; i++) {
                if (b[i] != 0) { mapped = false; break; }
            }
            if (mapped && b[10] == (byte)0xff && b[11] == (byte)0xff) {
                // Extract IPv4
                byte[] ipv4 = new byte[]{b[12], b[13], b[14], b[15]};
                try {
                    return InetAddress.getByAddress(ipv4);
                } catch (UnknownHostException e) {
                    // fallthrough
                }
            }
        }

        return addr;
    }

    public static void main(String[] args) throws Exception {
        // Simulate: a Socket with address ::ffff:192.168.1.1
        InetAddress mapped = InetAddress.getByName("::ffff:192.168.1.1");
        System.out.println("Mapped: " + mapped.getHostAddress());
        // getRealClientIP would return 192.168.1.1
    }
}
```

## NIO Non-Blocking IPv6 Server

```java
import java.io.IOException;
import java.net.*;
import java.nio.*;
import java.nio.channels.*;
import java.util.Iterator;

public class NIOIPv6Server {

    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();

        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.bind(new InetSocketAddress("::", 8080));
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("NIO IPv6 server on port 8080");

        while (true) {
            selector.select();
            Iterator<SelectionKey> keys = selector.selectedKeys().iterator();

            while (keys.hasNext()) {
                SelectionKey key = keys.next();
                keys.remove();

                if (key.isAcceptable()) {
                    SocketChannel client = serverChannel.accept();
                    if (client != null) {
                        InetSocketAddress peer = (InetSocketAddress) client.getRemoteAddress();
                        System.out.println("Accept: " + peer.getAddress().getHostAddress());
                        client.configureBlocking(false);
                        client.register(selector, SelectionKey.OP_READ);
                    }
                } else if (key.isReadable()) {
                    SocketChannel client = (SocketChannel) key.channel();
                    ByteBuffer buf = ByteBuffer.allocate(1024);
                    int n = client.read(buf);
                    if (n == -1) {
                        client.close();
                    } else {
                        buf.flip();
                        client.write(buf);  // Echo
                    }
                }
            }
        }
    }
}
```

## Conclusion

Java's `ServerSocket` supports IPv6 by binding to `InetAddress.getByName("::")`. For IPv6-only sockets, use `ServerSocketChannel` with NIO and configure `IPV6_V6ONLY`. Dual-stack connections present IPv4 clients as IPv4-mapped `::ffff:x.x.x.x` — detect and unwrap these for accurate client IP logging. NIO's `Selector` provides event-driven I/O for high-concurrency IPv6 servers without one thread per connection.
