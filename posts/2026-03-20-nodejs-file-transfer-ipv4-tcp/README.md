# How to Transfer Files over IPv4 TCP Sockets in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node.js, TCP, IPv4, File Transfer, Networking, Stream, Net Module

Description: Implement a simple file transfer protocol using Node.js TCP sockets over IPv4, with streaming support for large files and progress reporting.

## Introduction

Transferring files over raw TCP sockets teaches the fundamentals of binary data streaming, protocol framing, and backpressure handling. This implementation sends a length-prefixed header followed by the file data - a common approach for custom binary protocols.

## Protocol Design

```text
[4 bytes: filename length][filename][8 bytes: file size][file data...]
```

This framing allows the receiver to know how much data to expect before the file transfer begins.

## File Transfer Server

```javascript
// server.js
const net = require('net');
const fs = require('fs');
const path = require('path');

const PORT = 4000;
const SAVE_DIR = './received';

// Ensure save directory exists
fs.mkdirSync(SAVE_DIR, { recursive: true });

const server = net.createServer((socket) => {
  console.log(`Connection from ${socket.remoteAddress}:${socket.remotePort}`);
  
  let state = 'HEADER';  // State machine: HEADER -> FILENAME -> SIZE -> DATA
  let headerBuf = Buffer.alloc(0);
  let filename = '';
  let fileSize = 0;
  let bytesReceived = 0;
  let fileStream = null;
  let filenameLengthExpected = 4;
  let filenameExpected = 0;
  
  socket.on('data', (chunk) => {
    headerBuf = Buffer.concat([headerBuf, chunk]);
    
    while (headerBuf.length > 0) {
      if (state === 'HEADER') {
        // Wait for 4 bytes (filename length)
        if (headerBuf.length < 4) break;
        filenameExpected = headerBuf.readUInt32BE(0);
        headerBuf = headerBuf.subarray(4);
        state = 'FILENAME';
      }
      
      if (state === 'FILENAME') {
        if (headerBuf.length < filenameExpected) break;
        filename = headerBuf.subarray(0, filenameExpected).toString('utf8');
        filename = path.basename(filename); // Safety: strip path components
        headerBuf = headerBuf.subarray(filenameExpected);
        state = 'SIZE';
      }
      
      if (state === 'SIZE') {
        if (headerBuf.length < 8) break;
        fileSize = Number(headerBuf.readBigUInt64BE(0));
        headerBuf = headerBuf.subarray(8);
        state = 'DATA';
        
        const savePath = path.join(SAVE_DIR, filename);
        fileStream = fs.createWriteStream(savePath);
        bytesReceived = 0;
        console.log(`Receiving: ${filename} (${fileSize} bytes)`);
      }
      
      if (state === 'DATA') {
        if (headerBuf.length === 0) break;
        
        const remaining = fileSize - bytesReceived;
        const toWrite = headerBuf.subarray(0, Math.min(remaining, headerBuf.length));
        
        fileStream.write(toWrite);
        bytesReceived += toWrite.length;
        headerBuf = headerBuf.subarray(toWrite.length);
        
        const progress = ((bytesReceived / fileSize) * 100).toFixed(1);
        process.stdout.write(`\rProgress: ${progress}%`);
        
        if (bytesReceived >= fileSize) {
          fileStream.end();
          console.log(`\nSaved: ${path.join(SAVE_DIR, filename)}`);
          socket.write('OK\n');
          state = 'HEADER'; // Ready for next file
        }
      }
    }
  });
  
  socket.on('error', (err) => console.error(`Error: ${err.message}`));
  socket.on('end', () => {
    if (fileStream && !fileStream.destroyed) fileStream.end();
    console.log('Transfer complete');
  });
});

server.listen(PORT, '0.0.0.0', () => {
  console.log(`File server listening on port ${PORT}`);
});
```

## File Transfer Client

```javascript
// client.js
const net = require('net');
const fs = require('fs');
const path = require('path');

async function sendFile(serverIp, port, filePath) {
  const filename = path.basename(filePath);
  const stats = fs.statSync(filePath);
  const fileSize = BigInt(stats.size);
  
  console.log(`Sending ${filename} (${stats.size} bytes) to ${serverIp}:${port}`);
  
  return new Promise((resolve, reject) => {
    const socket = net.createConnection({ host: serverIp, port: port, family: 4 });
    
    socket.on('connect', () => {
      // Build header: [4 bytes filename len][filename][8 bytes file size]
      const filenameBuf = Buffer.from(filename, 'utf8');
      const header = Buffer.alloc(4 + filenameBuf.length + 8);
      
      header.writeUInt32BE(filenameBuf.length, 0);
      filenameBuf.copy(header, 4);
      header.writeBigUInt64BE(fileSize, 4 + filenameBuf.length);
      
      socket.write(header);
      
      // Stream the file
      const fileStream = fs.createReadStream(filePath);
      let bytesSent = 0;
      
      fileStream.on('data', (chunk) => {
        bytesSent += chunk.length;
        const progress = ((bytesSent / stats.size) * 100).toFixed(1);
        process.stdout.write(`\rSending: ${progress}%`);
        
        // Handle backpressure
        if (!socket.write(chunk)) {
          fileStream.pause();
          socket.once('drain', () => fileStream.resume());
        }
      });
      
      fileStream.on('end', () => {
        console.log('\nFile sent. Waiting for acknowledgment...');
      });
      
      fileStream.on('error', reject);
    });
    
    // Wait for server acknowledgment
    socket.on('data', (data) => {
      if (data.toString().trim() === 'OK') {
        console.log('Transfer confirmed by server');
        socket.end();
        resolve();
      }
    });
    
    socket.on('error', reject);
    socket.on('close', resolve);
  });
}

// Usage
const [,, serverIp, filePath] = process.argv;
if (!serverIp || !filePath) {
  console.error('Usage: node client.js <server-ip> <file-path>');
  process.exit(1);
}

sendFile(serverIp, 4000, filePath).catch(console.error);
```

## Running the Transfer

```bash
# Start the server

node server.js

# Send a file from another terminal (or machine)
node client.js 192.168.1.100 ./large-file.zip
```

## Conclusion

This file transfer implementation demonstrates TCP framing, binary protocol design, stream backpressure handling, and progress reporting - all fundamental skills for building custom binary protocols over IPv4 TCP in Node.js.
