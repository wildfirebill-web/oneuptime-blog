# How to Use Python Twisted for IPv6 Networking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Python, Twisted, Async Networking, TCP Server

Description: Build IPv6-capable asynchronous servers and clients using Python Twisted, including TCP and UDP servers, dual-stack endpoints, and protocol implementations.

## Install Twisted

```bash
pip install twisted
```

## IPv6 TCP Server

```python
from twisted.internet import reactor, protocol
from twisted.internet.endpoints import serverFromString

class IPv6EchoProtocol(protocol.Protocol):
    """Echo server that works over IPv6."""

    def connectionMade(self):
        peer = self.transport.getPeer()
        print(f"Connection from {peer.host}:{peer.port} (IPv{6 if ':' in peer.host else 4})")

    def dataReceived(self, data: bytes):
        # Echo back what we received
        self.transport.write(data)

    def connectionLost(self, reason):
        peer = self.transport.getPeer()
        print(f"Connection closed: {peer.host}")

class IPv6EchoFactory(protocol.ServerFactory):
    protocol = IPv6EchoProtocol

# Listen on IPv6 (:: binds to all IPv6 interfaces)

endpoint = serverFromString(reactor, "tcp6:port=8080:interface=\\:\\:")
d = endpoint.listen(IPv6EchoFactory())

print("IPv6 echo server listening on port 8080")
reactor.run()
```

## IPv6 TCP Client

```python
from twisted.internet import reactor, protocol, defer
from twisted.internet.endpoints import clientFromString

class IPv6ClientProtocol(protocol.Protocol):
    """TCP client that connects over IPv6."""

    def __init__(self, data_to_send: bytes):
        self.data = data_to_send
        self.response = b""

    def connectionMade(self):
        print(f"Connected to {self.transport.getPeer().host}")
        self.transport.write(self.data)

    def dataReceived(self, data: bytes):
        self.response += data
        print(f"Received: {data.decode()!r}")

    def connectionLost(self, reason):
        print("Connection closed")
        reactor.stop()

class IPv6ClientFactory(protocol.ClientFactory):
    def __init__(self, data: bytes):
        self.data = data

    def buildProtocol(self, addr):
        return IPv6ClientProtocol(self.data)

    def clientConnectionFailed(self, connector, reason):
        print(f"Connection failed: {reason.getErrorMessage()}")
        reactor.stop()

# Connect to IPv6 server - bracket notation in endpoint string
endpoint = clientFromString(reactor, "tcp6:host=2001\\:db8\\:\\:1:port=8080")
endpoint.connect(IPv6ClientFactory(b"Hello IPv6!"))
reactor.run()
```

## Dual-Stack Server (IPv4 + IPv6)

```python
from twisted.internet import reactor, protocol
from twisted.internet.endpoints import serverFromString
from twisted.application import service, internet

class ChatProtocol(protocol.Protocol):
    """Simple chat server protocol."""

    def connectionMade(self):
        self.factory.clients.add(self)
        peer = self.transport.getPeer()
        print(f"New client: {peer.host}")

    def dataReceived(self, data: bytes):
        # Broadcast to all clients
        for client in self.factory.clients:
            if client is not self:
                client.transport.write(data)

    def connectionLost(self, reason):
        self.factory.clients.discard(self)

class ChatFactory(protocol.ServerFactory):
    protocol = ChatProtocol

    def __init__(self):
        self.clients = set()

factory = ChatFactory()

# Listen on both IPv4 and IPv6 simultaneously
ipv6_endpoint = serverFromString(reactor, "tcp6:port=9000:interface=\\:\\:")
ipv4_endpoint = serverFromString(reactor, "tcp4:port=9000:interface=0.0.0.0")

# On dual-stack Linux with IPV6_V6ONLY=0, tcp6 :: also accepts IPv4
# Use explicit both for guaranteed dual-stack
ipv6_endpoint.listen(factory)
ipv4_endpoint.listen(factory)

print("Dual-stack chat server on port 9000")
reactor.run()
```

## IPv6 UDP Server

```python
from twisted.internet import reactor, protocol

class IPv6UDPProtocol(protocol.DatagramProtocol):
    """UDP server listening on IPv6."""

    def startProtocol(self):
        print(f"UDP server started on {self.transport.getHost()}")

    def datagramReceived(self, data: bytes, addr: tuple):
        host, port = addr[0], addr[1]
        print(f"Received {len(data)} bytes from [{host}]:{port}")
        # Echo back
        self.transport.write(data, addr)

# Bind to all IPv6 interfaces on UDP port 5000
reactor.listenUDP(5000, IPv6UDPProtocol(), interface="::")
print("IPv6 UDP server on port 5000")
reactor.run()
```

## Deferred IPv6 HTTP Client

```python
from twisted.internet import reactor
from twisted.web.client import Agent, readBody
from twisted.internet.endpoints import HostnameEndpoint
from twisted.web.http_headers import Headers

def fetch_ipv6_page(url: str):
    """Fetch a web page, preferring IPv6."""
    from twisted.web.client import Agent
    from twisted.internet import reactor

    agent = Agent(reactor)
    d = agent.request(
        b"GET",
        url.encode(),
        Headers({"User-Agent": ["Twisted IPv6 Client/1.0"]}),
    )

    def got_response(response):
        print(f"Status: {response.code}")
        print(f"Headers: {list(response.headers.getAllRawHeaders())[:3]}")
        return readBody(response)

    def got_body(body):
        print(f"Body length: {len(body)} bytes")
        reactor.stop()

    def handle_error(failure):
        print(f"Error: {failure.getErrorMessage()}")
        reactor.stop()

    d.addCallback(got_response)
    d.addCallback(got_body)
    d.addErrback(handle_error)

# Note: brackets required for IPv6 literal in URL
fetch_ipv6_page("http://[2001:4860:4860::8888]/")
reactor.run()
```

## Conclusion

Twisted uses endpoint strings to specify IPv6: `tcp6:port=8080:interface=\\:\\:` binds to all IPv6 interfaces, `tcp6:host=2001\\:db8\\:\\:1:port=80` connects to an IPv6 address (colons must be escaped as `\\:` in endpoint strings). For UDP, pass `interface="::"` to `reactor.listenUDP()`. Create dual-stack servers by listening on both `tcp4` and `tcp6` endpoints with the same factory. Twisted's `Agent` automatically resolves AAAA records for domain names, preferring IPv6 when available. Use Twisted for high-performance, event-driven IPv6 servers where async I/O and protocol composition (TLS, compression, framing) are needed.
