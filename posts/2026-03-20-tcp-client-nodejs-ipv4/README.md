# How to Build a TCP Client in Node.js for IPv4 Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node.js, TCP, Client, IPv4, Networking, net module

Description: Learn how to build a TCP client in Node.js that connects to an IPv4 server using the net module, sends data, and handles responses.

## Basic TCP Client

```javascript
const net = require('net');

const SERVER_HOST = '127.0.0.1';
const SERVER_PORT = 9000;

// Create and connect a TCP socket
const client = net.createConnection(
    { host: SERVER_HOST, port: SERVER_PORT, family: 4 },
    () => {
        // 'connect' callback: called when connection is established
        console.log(`Connected to ${SERVER_HOST}:${SERVER_PORT}`);

        // Send data to the server
        client.write('Hello, TCP server!\n');
    }
);

// Handle incoming data from the server
client.on('data', (data) => {
    console.log(`Server response: ${data.toString().trim()}`);
    // Close the connection after receiving the reply
    client.end();
});

// Connection closed
client.on('end', () => {
    console.log('Disconnected from server');
});

// Handle connection or communication errors
client.on('error', (err) => {
    console.error(`Connection error: ${err.message}`);
});
```

## Connecting with Timeout

```javascript
const net = require('net');

function connectWithTimeout(host, port, timeoutMs = 5000) {
    return new Promise((resolve, reject) => {
        const socket = new net.Socket();
        let connected = false;

        // Set connection timeout
        socket.setTimeout(timeoutMs);

        socket.connect({ host, port, family: 4 }, () => {
            connected = true;
            socket.setTimeout(0);  // Clear timeout once connected
            resolve(socket);
        });

        socket.on('timeout', () => {
            if (!connected) {
                socket.destroy();
                reject(new Error(`Connection to ${host}:${port} timed out`));
            }
        });

        socket.on('error', reject);
    });
}

async function main() {
    try {
        const socket = await connectWithTimeout('127.0.0.1', 9000, 3000);
        console.log('Connected!');

        // Use the socket
        socket.write('Hello!\n');

        socket.on('data', (data) => {
            console.log(`Response: ${data.toString()}`);
            socket.end();
        });
    } catch (err) {
        console.error(`Failed: ${err.message}`);
    }
}

main();
```

## Sending and Receiving Multiple Messages

```javascript
const net = require('net');

class TCPClient {
    constructor(host, port) {
        this.host = host;
        this.port = port;
        this.socket = null;
        this.buffer = '';
    }

    connect() {
        return new Promise((resolve, reject) => {
            this.socket = net.createConnection(
                { host: this.host, port: this.port, family: 4 },
                resolve
            );
            this.socket.on('error', reject);
        });
    }

    send(message) {
        return new Promise((resolve, reject) => {
            // Append newline as message delimiter
            this.socket.write(message + '\n');

            const onData = (data) => {
                this.buffer += data.toString();
                const lineEnd = this.buffer.indexOf('\n');
                if (lineEnd !== -1) {
                    const response = this.buffer.slice(0, lineEnd);
                    this.buffer = this.buffer.slice(lineEnd + 1);
                    this.socket.removeListener('data', onData);
                    resolve(response);
                }
            };

            this.socket.on('data', onData);
            this.socket.once('error', reject);
        });
    }

    close() {
        this.socket?.end();
    }
}

async function main() {
    const client = new TCPClient('127.0.0.1', 9000);
    await client.connect();

    for (const msg of ['hello', 'world', 'goodbye']) {
        const reply = await client.send(msg);
        console.log(`Sent: ${msg} | Got: ${reply}`);
    }

    client.close();
}

main().catch(console.error);
```

## Reconnect Logic

```javascript
const net = require('net');

function createReconnectingClient(host, port, onMessage) {
    let socket;
    let reconnectDelay = 1000;

    function connect() {
        socket = net.createConnection({ host, port, family: 4 });

        socket.on('connect', () => {
            console.log('Connected');
            reconnectDelay = 1000;  // Reset backoff on successful connect
        });

        socket.on('data', onMessage);

        socket.on('close', () => {
            console.log(`Disconnected. Reconnecting in ${reconnectDelay}ms...`);
            setTimeout(connect, reconnectDelay);
            reconnectDelay = Math.min(reconnectDelay * 2, 30000);  // Exponential backoff
        });

        socket.on('error', (err) => {
            console.error(`Socket error: ${err.message}`);
        });
    }

    connect();
    return { send: (data) => socket?.write(data) };
}

const client = createReconnectingClient('127.0.0.1', 9000, (data) => {
    console.log(`Received: ${data}`);
});
```

## Conclusion

Node.js TCP clients use `net.createConnection()` with `{ family: 4 }` to force IPv4. Handle `data`, `end`, and `error` events for communication and cleanup. For robust clients, implement connection timeouts and exponential backoff reconnection logic to handle transient server unavailability.
