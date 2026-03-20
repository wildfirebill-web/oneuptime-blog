# How to Configure IPv6 for Server Load Balancing in Data Centers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, Load Balancing, HAProxy, NGINX, Data Center, High Availability

Description: Configure IPv6 server load balancing in data centers using HAProxy and NGINX, including VIP setup and health check configuration.

## IPv6 Load Balancing Overview

Server load balancers (SLBs) in IPv6 data centers operate on Virtual IP (VIP) addresses. The load balancer accepts traffic on its IPv6 VIP and distributes it to backend servers. No NAT is needed between the VIP and backends if they're on the same network.

## HAProxy IPv6 Configuration

HAProxy natively supports IPv6 frontends and backends. Here is a complete HTTP load balancing configuration:

```
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    maxconn 50000

defaults
    mode http
    timeout connect 5s
    timeout client  30s
    timeout server  30s

# Frontend: listen on IPv6 VIP
frontend web_frontend
    bind [2001:db8:vip::1]:80
    bind [2001:db8:vip::1]:443 ssl crt /etc/ssl/certs/server.pem
    default_backend web_backends

# Backend: distribute to IPv6 application servers
backend web_backends
    balance roundrobin
    option httpchk GET /health
    server app1 [2001:db8:app::10]:8080 check
    server app2 [2001:db8:app::11]:8080 check
    server app3 [2001:db8:app::12]:8080 check
```

Note: IPv6 addresses in HAProxy configurations must be wrapped in square brackets.

## NGINX IPv6 Load Balancing

NGINX upstream blocks work with IPv6 addresses:

```nginx
# /etc/nginx/nginx.conf

upstream app_servers {
    least_conn;
    server [2001:db8:app::10]:8080;
    server [2001:db8:app::11]:8080;
    server [2001:db8:app::12]:8080;
}

server {
    # Listen on both IPv4 and IPv6
    listen [::]:80 ipv6only=off;
    listen [::]:443 ssl ipv6only=off;

    location / {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## Direct Server Return (DSR) with IPv6

DSR is efficient for high-traffic load balancing — backends respond directly to clients without traffic returning through the load balancer:

```bash
# On the load balancer: send traffic to backend with original VIP destination
ip6tables -t mangle -A PREROUTING -d 2001:db8:vip::1 -p tcp --dport 80 \
  -j TPROXY --tproxy-mark 0x1 --on-port 80

# On each backend server: add the VIP as a loopback alias
ip -6 addr add 2001:db8:vip::1/128 dev lo
```

## Health Checks

Ensure health checks use IPv6 endpoints. For HAProxy, the `check` keyword on backend servers triggers health probes automatically. Monitor check results:

```bash
# View HAProxy stats via socket
echo "show stat" | socat /var/run/haproxy/admin.sock stdio | cut -d',' -f1,2,18
```

## Anycast VIP for Multi-Site Load Balancing

For geographic load distribution, advertise the same VIP from multiple data centers via BGP anycast. Traffic is routed to the nearest data center automatically.

## Conclusion

IPv6 load balancing is straightforward with tools like HAProxy and NGINX. The key differences from IPv4 are the bracket notation for addresses in configuration files and the option to use DSR without NAT complexity. Plan your VIP prefix separately from backend server prefixes for clean firewall policies.
