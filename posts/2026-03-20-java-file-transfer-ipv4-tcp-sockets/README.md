# How to Transfer Files over IPv4 TCP Sockets in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Java, TCP, IPv4, File Transfer, Socket, Networking, I/O Streams

Description: Implement a file transfer server and client in Java using TCP sockets over IPv4 with buffered I/O streams and progress reporting.

## Introduction

Java's socket and stream APIs make file transfer straightforward. By wrapping socket streams in `DataOutputStream`/`DataInputStream`, you can send structured headers (filename, file size) followed by the raw file bytes — a simple but effective file transfer protocol.

## File Transfer Protocol

```
Client -> Server:
  [UTF string: filename]
  [long 8 bytes: file size]
  [byte[]: file data]

Server -> Client:
  [UTF string: "OK" or error message]
```

## File Transfer Server

```java
import java.io.*;
import java.net.*;
import java.nio.file.*;
import java.util.concurrent.*;

public class FileTransferServer {
    
    private static final int PORT = 5001;
    private static final String SAVE_DIR = "./received_files";
    
    public static void main(String[] args) throws IOException {
        Files.createDirectories(Paths.get(SAVE_DIR));
        
        ExecutorService executor = Executors.newCachedThreadPool();
        
        try (ServerSocket serverSocket = new ServerSocket(PORT, 50,
                InetAddress.getByName("0.0.0.0"))) {
            
            System.out.println("File server listening on port " + PORT);
            
            while (true) {
                Socket client = serverSocket.accept();
                executor.submit(() -> handleClient(client));
            }
        }
    }
    
    private static void handleClient(Socket socket) {
        String clientAddr = socket.getRemoteSocketAddress().toString();
        System.out.println("Connection from: " + clientAddr);
        
        try (
            DataInputStream in = new DataInputStream(
                new BufferedInputStream(socket.getInputStream()));
            DataOutputStream out = new DataOutputStream(
                new BufferedOutputStream(socket.getOutputStream()))
        ) {
            // Read the filename
            String filename = in.readUTF();
            // Strip path components for security
            filename = Paths.get(filename).getFileName().toString();
            
            // Read the file size
            long fileSize = in.readLong();
            System.out.printf("Receiving: %s (%,d bytes)%n", filename, fileSize);
            
            // Receive and write the file
            Path savePath = Paths.get(SAVE_DIR, filename);
            try (BufferedOutputStream fileOut = new BufferedOutputStream(
                    new FileOutputStream(savePath.toFile()))) {
                
                byte[] buffer = new byte[8192];
                long bytesReceived = 0;
                int bytesRead;
                
                while (bytesReceived < fileSize &&
                       (bytesRead = in.read(buffer, 0,
                           (int) Math.min(buffer.length, fileSize - bytesReceived))) != -1) {
                    
                    fileOut.write(buffer, 0, bytesRead);
                    bytesReceived += bytesRead;
                    
                    // Print progress every 5%
                    if (bytesReceived % (fileSize / 20) < buffer.length) {
                        double pct = (bytesReceived * 100.0) / fileSize;
                        System.out.printf("\rProgress: %.0f%%", pct);
                    }
                }
                
                System.out.println("\nSaved: " + savePath);
            }
            
            // Send acknowledgment
            out.writeUTF("OK:" + filename);
            out.flush();
            
        } catch (IOException e) {
            System.err.println("Error handling client " + clientAddr + ": " + e.getMessage());
        } finally {
            try { socket.close(); } catch (IOException ignored) {}
        }
    }
}
```

## File Transfer Client

```java
import java.io.*;
import java.net.*;
import java.nio.file.*;

public class FileTransferClient {
    
    public static void sendFile(String serverIp, int port, String filePath) throws IOException {
        Path file = Paths.get(filePath);
        long fileSize = Files.size(file);
        String filename = file.getFileName().toString();
        
        System.out.printf("Sending %s (%,d bytes) to %s:%d%n", filename, fileSize, serverIp, port);
        
        // Force IPv4 connection
        InetAddress serverAddr = InetAddress.getByName(serverIp);
        
        try (Socket socket = new Socket(serverAddr, port);
             DataOutputStream out = new DataOutputStream(
                 new BufferedOutputStream(socket.getOutputStream()));
             DataInputStream in = new DataInputStream(
                 new BufferedInputStream(socket.getInputStream()));
             BufferedInputStream fileIn = new BufferedInputStream(
                 new FileInputStream(file.toFile()))
        ) {
            // Send filename
            out.writeUTF(filename);
            
            // Send file size
            out.writeLong(fileSize);
            
            // Stream the file
            byte[] buffer = new byte[8192];
            long bytesSent = 0;
            int bytesRead;
            
            while ((bytesRead = fileIn.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);
                bytesSent += bytesRead;
                
                double pct = (bytesSent * 100.0) / fileSize;
                System.out.printf("\rSending: %.0f%%", pct);
            }
            
            out.flush();
            System.out.println("\nTransfer complete. Waiting for acknowledgment...");
            
            // Read server acknowledgment
            String response = in.readUTF();
            System.out.println("Server response: " + response);
        }
    }
    
    public static void main(String[] args) throws IOException {
        if (args.length != 3) {
            System.err.println("Usage: FileTransferClient <server-ip> <port> <file-path>");
            System.exit(1);
        }
        
        sendFile(args[0], Integer.parseInt(args[1]), args[2]);
    }
}
```

## Compiling and Running

```bash
# Compile
javac FileTransferServer.java FileTransferClient.java

# Start server
java FileTransferServer

# Send a file (from same or different machine)
java FileTransferClient 192.168.1.100 5001 /path/to/large-file.zip
```

## Conclusion

Java's `DataOutputStream`/`DataInputStream` wrapping around `BufferedInputStream`/`BufferedOutputStream` provides an efficient, structured approach to file transfer over IPv4 TCP. Buffering reduces system call overhead, `DataOutputStream.writeUTF` handles the filename header cleanly, and progress reporting makes the transfer observable.
