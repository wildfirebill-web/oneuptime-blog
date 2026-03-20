# How to Use IPv6 with Java DatagramSocket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, IPv6, UDP, DatagramSocket, Multicast, Networking

Description: Use Java DatagramSocket and MulticastSocket for IPv6 UDP communication including unicast messaging, multicast groups, and NIO UDP channels.

## Basic IPv6 UDP Server

```java
import java.net.*;

public class IPv6UDPServer {

    public static void main(String[] args) throws Exception {
        // Bind to [::]:9000 for IPv6 UDP
        DatagramSocket socket = new DatagramSocket(
            new InetSocketAddress("::", 9000));

        System.out.println("UDP server on " + socket.getLocalSocketAddress());

        byte[] buf = new byte[1500];
        while (true) {
            DatagramPacket packet = new DatagramPacket(buf, buf.length);
            socket.receive(packet);

            String msg = new String(packet.getData(), 0, packet.getLength());
            System.out.printf("From %s: %s%n",
                packet.getAddress().getHostAddress(), msg);

            // Echo back
            byte[] reply = ("Echo: " + msg).getBytes();
            DatagramPacket response = new DatagramPacket(
                reply, reply.length,
                packet.getAddress(), packet.getPort());
            socket.send(response);
        }
    }
}
```

## IPv6 UDP Client

```java
import java.net.*;

public class IPv6UDPClient {

    public static void main(String[] args) throws Exception {
        // Bind to ephemeral port
        DatagramSocket socket = new DatagramSocket(
            new InetSocketAddress("::", 0));

        InetAddress server = InetAddress.getByName("2001:db8::1");
        byte[] data = "Hello IPv6 UDP".getBytes();

        DatagramPacket packet = new DatagramPacket(data, data.length, server, 9000);
        socket.send(packet);
        System.out.println("Sent to " + server.getHostAddress() + ":9000");

        // Receive response
        socket.setSoTimeout(5000);
        byte[] buf = new byte[1500];
        DatagramPacket response = new DatagramPacket(buf, buf.length);
        socket.receive(response);

        System.out.println("Response: " +
            new String(response.getData(), 0, response.getLength()));

        socket.close();
    }
}
```

## IPv6 Multicast with MulticastSocket

```java
import java.net.*;

public class IPv6MulticastReceiver {

    public static void main(String[] args) throws Exception {
        // ff02::fb is the mDNS multicast address (link-local scope)
        InetAddress group = InetAddress.getByName("ff02::fb");
        NetworkInterface iface = NetworkInterface.getByName("eth0");

        MulticastSocket socket = new MulticastSocket(5353);

        // Join the multicast group on the specified interface
        SocketAddress groupAddr = new InetSocketAddress(group, 5353);
        socket.joinGroup(groupAddr, iface);
        System.out.println("Joined ff02::fb on eth0");

        byte[] buf = new byte[1500];
        for (int i = 0; i < 10; i++) {
            DatagramPacket packet = new DatagramPacket(buf, buf.length);
            socket.receive(packet);
            System.out.printf("Multicast from %s: %d bytes%n",
                packet.getAddress().getHostAddress(), packet.getLength());
        }

        socket.leaveGroup(groupAddr, iface);
        socket.close();
    }
}
```

## Sending IPv6 Multicast

```java
import java.net.*;

public class IPv6MulticastSender {

    public static void main(String[] args) throws Exception {
        MulticastSocket socket = new MulticastSocket();

        // Set outgoing interface for multicast
        NetworkInterface iface = NetworkInterface.getByName("eth0");
        socket.setNetworkInterface(iface);
        socket.setTimeToLive(5);  // Max 5 hops

        InetAddress group = InetAddress.getByName("ff02::1");
        byte[] data = "Announcement".getBytes();

        DatagramPacket packet = new DatagramPacket(data, data.length, group, 5000);
        socket.send(packet);

        System.out.println("Sent multicast to ff02::1");
        socket.close();
    }
}
```

## NIO DatagramChannel for IPv6

```java
import java.net.*;
import java.nio.*;
import java.nio.channels.*;

public class NIOIPv6UDP {

    public static void main(String[] args) throws Exception {
        DatagramChannel channel = DatagramChannel.open(StandardProtocolFamily.INET6);
        channel.bind(new InetSocketAddress("::", 9000));
        channel.configureBlocking(true);

        System.out.println("NIO IPv6 UDP on " + channel.getLocalAddress());

        ByteBuffer buf = ByteBuffer.allocate(4096);
        while (true) {
            buf.clear();
            SocketAddress sender = channel.receive(buf);
            buf.flip();

            byte[] data = new byte[buf.remaining()];
            buf.get(data);
            System.out.printf("From %s: %s%n", sender, new String(data));

            // Echo
            buf.rewind();
            channel.send(buf, sender);
        }
    }
}
```

## Conclusion

Java's `DatagramSocket` works with IPv6 by binding to `new InetSocketAddress("::", port)`. For multicast, `MulticastSocket.joinGroup(SocketAddress, NetworkInterface)` is the Java 17+ API (the older `joinGroup(InetAddress)` is deprecated for IPv6). NIO's `DatagramChannel.open(StandardProtocolFamily.INET6)` creates IPv6-specific channels. Always specify `StandardProtocolFamily.INET6` for `DatagramChannel` to ensure the channel uses IPv6 rather than defaulting to IPv4.
