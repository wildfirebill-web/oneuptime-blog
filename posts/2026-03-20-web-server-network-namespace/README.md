# How to Run a Web Server Inside a Network Namespace

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Linux, Network Namespaces, Web Server, Nginx, Networking, Containers, iproute2

Description: Start a web server inside a Linux network namespace and access it from the host using veth pairs and port forwarding for isolated service testing.

## Introduction

Running a web server inside a network namespace simulates container-like isolation without a container runtime. The web server only has access to the network interfaces in its namespace, making it useful for testing isolated services, simulating multi-tenant environments, and understanding how container networking works under the hood.

## Prerequisites

- A configured namespace with a veth pair and IP address
- A web server installed (nginx, Apache, or Python's built-in server)
- Root access

## Quick Start: Python Web Server

The simplest approach uses Python's built-in HTTP server:

```bash
# Create and configure the namespace

ip netns add webns
ip link add veth-host type veth peer name veth-web
ip link set veth-web netns webns

# Configure host side
ip addr add 10.1.0.1/24 dev veth-host
ip link set veth-host up

# Configure namespace side
ip netns exec webns ip link set lo up
ip netns exec webns ip addr add 10.1.0.2/24 dev veth-web
ip netns exec webns ip link set veth-web up

# Start Python web server inside the namespace
ip netns exec webns python3 -m http.server 8080 &

# Access from the host
curl http://10.1.0.2:8080
```

## Run Nginx Inside a Namespace

For production-grade testing, run Nginx inside the namespace:

```bash
# Create a minimal nginx config for the namespace
mkdir -p /tmp/nginx-ns/{conf,html,logs}
cat > /tmp/nginx-ns/conf/nginx.conf << 'EOF'
worker_processes 1;
error_log /tmp/nginx-ns/logs/error.log;
pid /tmp/nginx-ns/logs/nginx.pid;

events { worker_connections 1024; }

http {
    access_log /tmp/nginx-ns/logs/access.log;

    server {
        # Bind to the namespace's IP address
        listen 10.1.0.2:80;
        root /tmp/nginx-ns/html;

        location / {
            index index.html;
        }
    }
}
EOF

# Create a test page
echo "<h1>Hello from namespace webns</h1>" > /tmp/nginx-ns/html/index.html

# Start nginx inside the namespace
ip netns exec webns nginx -c /tmp/nginx-ns/conf/nginx.conf

# Test from the host
curl http://10.1.0.2
```

## Access the Namespace Web Server via Port Forwarding

To access the namespace web server on the host's public IP:

```bash
# Forward host port 8080 to namespace port 80 using iptables
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.1.0.2:80
iptables -A FORWARD -p tcp -d 10.1.0.2 --dport 80 -j ACCEPT
```

## Stop the Web Server

```bash
# Kill the Python server
kill $(ip netns pids webns)

# Stop nginx
ip netns exec webns nginx -c /tmp/nginx-ns/conf/nginx.conf -s stop

# Clean up
ip netns delete webns
ip link delete veth-host
```

## Full Setup and Teardown Script

```bash
#!/bin/bash
# ns-webserver.sh: Start a web server in a namespace

NS=webns
HOST_IP=10.1.0.1/24
NS_IP=10.1.0.2

# Setup
ip netns add $NS
ip link add veth-host type veth peer name veth-web
ip link set veth-web netns $NS
ip addr add $HOST_IP dev veth-host && ip link set veth-host up
ip netns exec $NS ip link set lo up
ip netns exec $NS ip addr add ${NS_IP}/24 dev veth-web
ip netns exec $NS ip link set veth-web up

# Start web server
ip netns exec $NS python3 -m http.server 8080 &
echo "Web server started. Access at http://${NS_IP}:8080"

# Test
sleep 1
curl -s http://${NS_IP}:8080/ | head -5

echo "Press Enter to stop and clean up..."
read

# Teardown
kill $(ip netns pids $NS) 2>/dev/null
ip netns delete $NS
ip link delete veth-host 2>/dev/null
```

## Conclusion

Running a web server inside a network namespace demonstrates how container networking provides isolation while still allowing controlled access from the host. The veth pair provides connectivity, and the server binds to the namespace IP. Port forwarding rules on the host can expose the service externally without the server being aware it is isolated.
