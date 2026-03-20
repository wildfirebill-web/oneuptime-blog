# How to Implement IPv4 Multicast for Group Communication in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, IPv4, Multicast, UDP, Networking, MulticastSocket

Description: Learn how to implement IPv4 multicast group communication in Java using MulticastSocket, with sender and receiver examples, network interface binding, and group membership management.

## Multicast Sender

```java
import java.io.IOException;
import java.net.*;

public class MulticastSender {
    static final String GROUP_ADDR = "239.255.0.1";
    static final int    PORT       = 5007;
    static final int    TTL        = 2;  // number of network hops

    public static void main(String[] args) throws IOException, InterruptedException {
        InetAddress group = InetAddress.getByName(GROUP_ADDR);

        try (MulticastSocket socket = new MulticastSocket()) {
            socket.setTimeToLive(TTL);

            System.out.printf("Sending multicast to %s:%d%n", GROUP_ADDR, PORT);

            for (int i = 0; i < 10; i++) {
                String text = String.format(
                    "{\"seq\":%d, \"msg\":\"Hello group #%d\"}", i, i);
                byte[] buf  = text.getBytes();
                DatagramPacket packet = new DatagramPacket(buf, buf.length, group, PORT);
                socket.send(packet);
                System.out.println("Sent: " + text);
                Thread.sleep(1000);
            }
        }
    }
}
```

## Multicast Receiver

```java
import java.io.IOException;
import java.net.*;

public class MulticastReceiver {
    static final String GROUP_ADDR = "239.255.0.1";
    static final int    PORT       = 5007;

    public static void main(String[] args) throws IOException {
        InetAddress group   = InetAddress.getByName(GROUP_ADDR);
        byte[]      buf     = new byte[4096];

        try (MulticastSocket socket = new MulticastSocket(PORT)) {
            // Join the multicast group
            socket.joinGroup(group);
            System.out.printf("Joined %s:%d%n", GROUP_ADDR, PORT);

            try {
                while (true) {
                    DatagramPacket packet = new DatagramPacket(buf, buf.length);
                    socket.receive(packet);
                    String msg = new String(packet.getData(), 0, packet.getLength());
                    System.out.printf("From %s: %s%n",
                        packet.getAddress().getHostAddress(), msg);
                }
            } finally {
                socket.leaveGroup(group);
                System.out.println("Left multicast group");
            }
        }
    }
}
```

## Binding to a Specific Network Interface

```java
import java.net.*;

// Use the newer joinGroup(SocketAddress, NetworkInterface) API
public class InterfacedReceiver {
    public static void main(String[] args) throws Exception {
        String groupAddr = "239.255.0.1";
        int    port      = 5007;

        // Find the interface to use
        NetworkInterface ni = NetworkInterface.getByName("eth0");

        InetSocketAddress groupSock = new InetSocketAddress(
            InetAddress.getByName(groupAddr), port);

        try (MulticastSocket socket = new MulticastSocket(port)) {
            socket.joinGroup(groupSock, ni);
            System.out.println("Joined on interface: " + ni.getDisplayName());

            byte[] buf = new byte[4096];
            DatagramPacket pkt = new DatagramPacket(buf, buf.length);
            socket.receive(pkt);
            System.out.println(new String(pkt.getData(), 0, pkt.getLength()));

            socket.leaveGroup(groupSock, ni);
        }
    }
}
```

## Using NIO DatagramChannel for Multicast

```java
import java.net.*;
import java.nio.ByteBuffer;
import java.nio.channels.*;

public class NioMulticastReceiver {
    public static void main(String[] args) throws Exception {
        NetworkInterface ni    = NetworkInterface.getByName("eth0");
        InetAddress      group = InetAddress.getByName("239.255.0.1");

        DatagramChannel ch = DatagramChannel.open(StandardProtocolFamily.INET);
        ch.setOption(StandardSocketOptions.SO_REUSEADDR, true);
        ch.bind(new InetSocketAddress(5007));
        ch.setOption(StandardSocketOptions.IP_MULTICAST_IF, ni);
        MembershipKey key = ch.join(group, ni);

        System.out.println("NIO multicast receiver ready");

        ByteBuffer buf = ByteBuffer.allocate(4096);
        while (true) {
            buf.clear();
            SocketAddress src = ch.receive(buf);
            buf.flip();
            byte[] data = new byte[buf.remaining()];
            buf.get(data);
            System.out.printf("From %s: %s%n", src, new String(data));
        }
    }
}
```

## Conclusion

`MulticastSocket` provides a straightforward API for IPv4 multicast in Java: call `joinGroup(addr)` to receive packets and `leaveGroup(addr)` on shutdown. Use the newer `joinGroup(SocketAddress, NetworkInterface)` overload when you need to bind to a specific NIC (important on multi-homed hosts). Set `setTimeToLive(1)` for LAN-only delivery. For non-blocking multicast I/O at high throughput, use NIO `DatagramChannel` with `StandardProtocolFamily.INET` and the `join(group, interface)` method.
