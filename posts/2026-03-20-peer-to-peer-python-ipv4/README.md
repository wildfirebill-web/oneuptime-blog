# How to Implement Peer-to-Peer Communication over IPv4 in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv4, Peer-to-Peer, TCP, Networking, asyncio

Description: Learn how to implement peer-to-peer communication over IPv4 in Python, where each node acts as both a server and client, enabling direct bidirectional communication without a central server.

## P2P Node: Server + Client in One

```python
import asyncio
import json
import logging
from dataclasses import dataclass, field

log = logging.getLogger(__name__)

@dataclass
class Peer:
    host: str
    port: int
    writer: asyncio.StreamWriter

peers: dict[str, Peer] = {}   # "host:port" → Peer

async def handle_incoming(reader: asyncio.StreamReader,
                           writer: asyncio.StreamWriter) -> None:
    addr = writer.get_extra_info("peername")
    peer_id = f"{addr[0]}:{addr[1]}"
    log.info("[+] Incoming peer: %s", peer_id)
    peers[peer_id] = Peer(addr[0], addr[1], writer)

    try:
        while True:
            data = await reader.readline()
            if not data:
                break
            msg = json.loads(data.decode())
            log.info("Message from %s: %s", peer_id, msg)
            # Echo or process message
    except asyncio.IncompleteReadError:
        pass
    finally:
        del peers[peer_id]
        log.info("[-] Peer disconnected: %s", peer_id)
        writer.close()

async def connect_to_peer(host: str, port: int) -> None:
    """Initiate an outbound connection to another peer."""
    try:
        reader, writer = await asyncio.open_connection(host, port)
        peer_id = f"{host}:{port}"
        peers[peer_id] = Peer(host, port, writer)
        log.info("[→] Connected to %s", peer_id)

        asyncio.create_task(handle_incoming(reader, writer))
    except ConnectionRefusedError:
        log.warning("Peer %s:%d refused connection", host, port)

async def broadcast(message: dict) -> None:
    """Send a message to all connected peers."""
    payload = (json.dumps(message) + "\n").encode()
    dead = []
    for peer_id, peer in peers.items():
        try:
            peer.writer.write(payload)
            await peer.writer.drain()
        except Exception:
            dead.append(peer_id)
    for peer_id in dead:
        del peers[peer_id]

async def main(listen_port: int, known_peers: list[tuple[str, int]]) -> None:
    server = await asyncio.start_server(
        handle_incoming, "0.0.0.0", listen_port
    )
    log.info("P2P node listening on 0.0.0.0:%d", listen_port)

    # Connect to known peers
    for host, port in known_peers:
        await connect_to_peer(host, port)

    # Periodically announce presence
    async def announce():
        while True:
            await asyncio.sleep(10)
            await broadcast({"type": "ping", "port": listen_port})

    async with server:
        await asyncio.gather(server.serve_forever(), announce())
```

## Usage: Two Nodes

```bash
# Node 1: port 9001, no known peers yet
python node.py --port 9001

# Node 2: port 9002, knows about Node 1
python node.py --port 9002 --peer 127.0.0.1:9001
```

## Simple Message Exchange

```python
import asyncio

async def p2p_demo():
    # Start two nodes
    async def node(port, peer_port):
        server = await asyncio.start_server(
            lambda r, w: asyncio.create_task(chat(r, w)),
            "127.0.0.1", port
        )
        async with server:
            await asyncio.sleep(0.5)  # let both start
            reader, writer = await asyncio.open_connection("127.0.0.1", peer_port)
            writer.write(f"Hello from :{port}\n".encode())
            await writer.drain()
            reply = await reader.readline()
            print(f"Node {port} received: {reply.decode().strip()}")
            writer.close()

    async def chat(reader, writer):
        data = await reader.readline()
        writer.write(b"ACK: " + data)
        await writer.drain()
        writer.close()

    await asyncio.gather(node(9001, 9002), node(9002, 9001))

asyncio.run(p2p_demo())
```

## Conclusion

A P2P node combines a server (listening for inbound peers) with a client (connecting to known peers). Use `asyncio.start_server` + `asyncio.open_connection` in the same process to achieve both roles concurrently. Store active connections in a dictionary for broadcasting. Handle disconnections in `finally` blocks to keep the peer registry accurate. For peer discovery, start with a bootstrap peer list and exchange peer addresses in your application protocol.
