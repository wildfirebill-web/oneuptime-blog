# How to Use Java NIO Channels for IPv4 Socket Programming

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, NIO, Channels, IPv4, Sockets, Non-Blocking, Networking

Description: Learn how to use Java NIO's ServerSocketChannel and SocketChannel for high-performance IPv4 socket programming with non-blocking I/O.

## Why Java NIO?

Traditional `java.net` sockets are blocking—each connection needs its own thread. Java NIO (Non-blocking I/O) channels allow a single thread to handle thousands of connections using a `Selector`, similar to `epoll` on Linux.

## Basic NIO TCP Server

```java
import java.io.*;
import java.net.*;
import java.nio.*;
import java.nio.channels.*;
import java.util.*;

public class NioTcpServer {
    private static final int PORT = 9000;

    public static void main(String[] args) throws IOException {
        // Open a non-blocking server socket channel
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.bind(new InetSocketAddress("0.0.0.0", PORT));
        serverChannel.setOption(StandardSocketOptions.SO_REUSEADDR, true);

        // Create a Selector to monitor multiple channels
        Selector selector = Selector.open();

        // Register the server channel for ACCEPT events
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("NIO server listening on port " + PORT);

        ByteBuffer buffer = ByteBuffer.allocate(4096);

        while (true) {
            // Block until at least one channel is ready
            selector.select();

            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> iter = selectedKeys.iterator();

            while (iter.hasNext()) {
                SelectionKey key = iter.next();
                iter.remove();

                if (key.isAcceptable()) {
                    // Accept new connection
                    SocketChannel client = serverChannel.accept();
                    client.configureBlocking(false);
                    client.register(selector, SelectionKey.OP_READ);
                    System.out.println("Accepted: " + client.getRemoteAddress());

                } else if (key.isReadable()) {
                    // Read data from client
                    SocketChannel client = (SocketChannel) key.channel();
                    buffer.clear();
                    int bytesRead = client.read(buffer);

                    if (bytesRead == -1) {
                        // Client closed connection
                        System.out.println("Client disconnected: " + client.getRemoteAddress());
                        key.cancel();
                        client.close();
                    } else {
                        // Echo back: flip switches buffer from write to read mode
                        buffer.flip();
                        client.write(buffer);
                    }
                }
            }
        }
    }
}
```

## NIO TCP Client

```java
import java.io.*;
import java.net.*;
import java.nio.*;
import java.nio.channels.*;
import java.nio.charset.*;

public class NioTcpClient {
    public static void main(String[] args) throws IOException {
        // Open a SocketChannel
        SocketChannel channel = SocketChannel.open();
        channel.configureBlocking(true);  // Blocking mode for simple client

        // Connect to server
        boolean connected = channel.connect(new InetSocketAddress("127.0.0.1", 9000));
        if (!connected) {
            channel.finishConnect();
        }
        System.out.println("Connected: " + channel.getRemoteAddress());

        // Send message
        String message = "Hello from NIO client!\n";
        ByteBuffer writeBuffer = ByteBuffer.wrap(message.getBytes(StandardCharsets.UTF_8));
        channel.write(writeBuffer);

        // Read response
        ByteBuffer readBuffer = ByteBuffer.allocate(1024);
        int bytesRead = channel.read(readBuffer);
        readBuffer.flip();

        String response = StandardCharsets.UTF_8.decode(readBuffer).toString();
        System.out.println("Response: " + response.trim());

        channel.close();
    }
}
```

## Reading and Writing with Buffers

```java
// Write a string to a channel
public static void writeString(SocketChannel ch, String msg) throws IOException {
    ByteBuffer buf = ByteBuffer.wrap(msg.getBytes(StandardCharsets.UTF_8));
    while (buf.hasRemaining()) {
        ch.write(buf);
    }
}

// Read a specific number of bytes from a channel
public static byte[] readBytes(SocketChannel ch, int count) throws IOException {
    ByteBuffer buf = ByteBuffer.allocate(count);
    while (buf.hasRemaining()) {
        int n = ch.read(buf);
        if (n == -1) throw new EOFException("Channel closed");
    }
    return buf.array();
}
```

## Setting Channel Socket Options

```java
ServerSocketChannel server = ServerSocketChannel.open();

// Java NIO socket options
server.setOption(StandardSocketOptions.SO_REUSEADDR, true);
server.setOption(StandardSocketOptions.SO_REUSEPORT, true);

SocketChannel client = server.accept();
client.setOption(StandardSocketOptions.TCP_NODELAY, true);    // Disable Nagle
client.setOption(StandardSocketOptions.SO_KEEPALIVE, true);   // TCP keepalive
client.setOption(StandardSocketOptions.SO_SNDBUF, 65536);     // Send buffer
client.setOption(StandardSocketOptions.SO_RCVBUF, 65536);     // Recv buffer
```

## Conclusion

Java NIO channels with `Selector` enable event-driven I/O where a single thread multiplexes thousands of connections—similar to Node.js's event loop. The key operations are registering channels with `OP_ACCEPT` and `OP_READ` interest sets, then reacting to ready events in the selector loop. For new Java projects, consider `java.nio.channels.AsynchronousServerSocketChannel` (AIO) or Netty/Vert.x frameworks for higher-level abstractions.
