# How to Create a UDP Server with Node.js dgram Module on IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node.js, UDP, Dgram, IPv4, Networking, Datagram

Description: Learn how to create a UDP server and client in Node.js using the built-in dgram module for IPv4 datagram communication.

## Basic UDP Server

```javascript
const dgram = require('dgram');

// 'udp4' creates an IPv4 UDP socket
const server = dgram.createSocket('udp4');

server.on('message', (msg, rinfo) => {
    console.log(`Received ${msg.length} bytes from ${rinfo.address}:${rinfo.port}`);
    console.log(`Message: ${msg.toString()}`);

    // Send a reply back to the sender
    const reply = Buffer.from(`Echo: ${msg.toString()}`);
    server.send(reply, rinfo.port, rinfo.address, (err) => {
        if (err) {
            console.error(`Reply error: ${err.message}`);
        }
    });
});

server.on('error', (err) => {
    console.error(`Server error: ${err.message}`);
    server.close();
});

server.on('listening', () => {
    const addr = server.address();
    console.log(`UDP server listening on ${addr.address}:${addr.port}`);
});

// Bind to a specific port on all IPv4 interfaces
server.bind(9001, '0.0.0.0');
```

## UDP Client

```javascript
const dgram = require('dgram');

const SERVER_HOST = '127.0.0.1';
const SERVER_PORT = 9001;

const client = dgram.createSocket('udp4');

// Set a timeout to receive response
const timeout = setTimeout(() => {
    console.error('No response received (timeout)');
    client.close();
}, 3000);

client.on('message', (msg, rinfo) => {
    clearTimeout(timeout);
    console.log(`Response from ${rinfo.address}:${rinfo.port}: ${msg.toString()}`);
    client.close();
});

client.on('error', (err) => {
    clearTimeout(timeout);
    console.error(`Client error: ${err.message}`);
    client.close();
});

// Send a UDP datagram
const message = Buffer.from('Hello, UDP server!');
client.send(message, SERVER_PORT, SERVER_HOST, (err) => {
    if (err) {
        console.error(`Send error: ${err.message}`);
        clearTimeout(timeout);
        client.close();
    } else {
        console.log(`Sent ${message.length} bytes to ${SERVER_HOST}:${SERVER_PORT}`);
    }
});
```

## Sending Multiple Datagrams

```javascript
const dgram = require('dgram');

const client = dgram.createSocket('udp4');
const SERVER = { host: '127.0.0.1', port: 9001 };

// Promisify UDP send
function sendDatagram(socket, data, port, host) {
    return new Promise((resolve, reject) => {
        socket.send(Buffer.from(data), port, host, (err) => {
            if (err) reject(err);
            else resolve();
        });
    });
}

async function main() {
    const messages = ['ping', 'status', 'metrics'];

    for (const msg of messages) {
        await sendDatagram(client, msg, SERVER.port, SERVER.host);
        console.log(`Sent: ${msg}`);
        await new Promise(r => setTimeout(r, 100));  // Small delay between datagrams
    }

    client.close();
}

main().catch(console.error);
```

## Handling Server Socket Events

```javascript
const dgram = require('dgram');

const server = dgram.createSocket('udp4');

server.on('listening', () => {
    console.log(`Listening: ${JSON.stringify(server.address())}`);
    // Get socket details
    console.log(`Family: ${server.address().family}`);
});

server.on('message', (msg, rinfo) => {
    // rinfo.address: sender IPv4 address
    // rinfo.port: sender port
    // rinfo.family: 'IPv4' or 'IPv6'
    // rinfo.size: byte length of datagram
    console.log(`From: ${rinfo.address}:${rinfo.port} family=${rinfo.family} size=${rinfo.size}`);
    console.log(`Data: ${msg.toString('utf8')}`);
});

server.on('close', () => {
    console.log('Server socket closed');
});

server.on('error', (err) => {
    console.error(`Error: ${err.stack}`);
});

server.bind(9001);

// Graceful shutdown
process.on('SIGINT', () => {
    console.log('Closing server...');
    server.close();
});
```

## Conclusion

Node.js UDP sockets use `dgram.createSocket('udp4')` for IPv4 datagrams. Servers bind with `server.bind(port, host)` and receive via the `message` event which provides the `rinfo` object with sender address. Clients send with `client.send(buffer, port, host, callback)`. Always handle the `error` event to prevent unhandled exception crashes.
