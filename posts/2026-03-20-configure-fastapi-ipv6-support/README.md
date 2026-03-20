# How to Configure FastAPI for IPv6 Support

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: FastAPI, Python, IPv6, Uvicorn, ASGI, Dual-Stack, Pydantic

Description: Configure FastAPI with Uvicorn to listen on IPv6, extract client IPv6 addresses from requests, validate IPv6 inputs with Pydantic, and deploy behind an IPv6 proxy.

## Introduction

FastAPI uses Uvicorn (ASGI server) which has native IPv6 support. Binding to `::` enables dual-stack operation. FastAPI's Pydantic integration makes validating IPv6 address inputs straightforward.

## Step 1: Run FastAPI on IPv6

```python
# main.py
import uvicorn
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/")
async def root(request: Request):
    return {"client": request.client.host}

if __name__ == "__main__":
    # Listen on all IPv6 (and IPv4 via dual-stack)
    uvicorn.run("main:app", host="::", port=8000, reload=True)
```

```bash
# Start directly
uvicorn main:app --host "::" --port 8000

# IPv6-only
uvicorn main:app --host "::" --port 8000 --no-access-log

# Test
curl -6 http://[::1]:8000/
```

## Step 2: Extract Client IPv6 Address

```python
# middleware/client_ip.py
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
import ipaddress

class ClientIPMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Get real IP from X-Forwarded-For
        xff = request.headers.get("X-Forwarded-For")
        if xff:
            ip = xff.split(",")[0].strip()
        else:
            ip = request.client.host if request.client else "unknown"

        # Normalize IPv4-mapped IPv6
        if ip.startswith("::ffff:"):
            ip = ip[7:]

        # Attach to request state
        request.state.client_ip = ip
        request.state.is_ipv6 = False
        try:
            addr = ipaddress.ip_address(ip)
            request.state.is_ipv6 = (addr.version == 6)
        except ValueError:
            pass

        return await call_next(request)

app = FastAPI()
app.add_middleware(ClientIPMiddleware)
```

## Step 3: IPv6 Input Validation with Pydantic

```python
# schemas.py
from pydantic import BaseModel, field_validator
import ipaddress

class NetworkEndpoint(BaseModel):
    address: str
    port: int

    @field_validator("address")
    @classmethod
    def validate_ipv6(cls, v: str) -> str:
        try:
            addr = ipaddress.ip_address(v)
        except ValueError:
            raise ValueError(f"Invalid IP address: {v}")
        if addr.is_loopback:
            raise ValueError("Loopback addresses not allowed")
        return str(addr)

    def url(self) -> str:
        """Return properly formatted URL for IPv6."""
        addr = ipaddress.ip_address(self.address)
        if addr.version == 6:
            return f"http://[{self.address}]:{self.port}"
        return f"http://{self.address}:{self.port}"
```

```python
# routes/network.py
from fastapi import APIRouter, HTTPException
from schemas import NetworkEndpoint

router = APIRouter()

@router.post("/endpoint")
async def create_endpoint(endpoint: NetworkEndpoint):
    return {
        "address": endpoint.address,
        "port": endpoint.port,
        "url": endpoint.url(),
    }
```

## Step 4: Rate Limiting by IPv6 /64 Subnet

```python
# middleware/rate_limit.py
from slowapi import Limiter
from slowapi.util import get_remote_address
import ipaddress

def get_ipv6_subnet_key(request: Request) -> str:
    """Use /64 subnet as rate limit key for IPv6."""
    ip = request.state.client_ip if hasattr(request.state, "client_ip") \
        else get_remote_address(request)
    try:
        addr = ipaddress.ip_address(ip)
        if addr.version == 6:
            net = ipaddress.IPv6Network(f"{ip}/64", strict=False)
            return str(net.network_address)
    except ValueError:
        pass
    return ip

limiter = Limiter(key_func=get_ipv6_subnet_key)
app.state.limiter = limiter
```

## Step 5: Production Deployment

```bash
# Gunicorn with Uvicorn workers (production)
gunicorn main:app \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind "[::]:8000" \
    --workers 4

# With both IPv4 and IPv6
gunicorn main:app \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind "0.0.0.0:8000" \
    --bind "[::]:8000" \
    --workers 4
```

## Conclusion

FastAPI on IPv6 requires passing `host="::"` to Uvicorn and configuring middleware to extract client IPv6 addresses from proxy headers. Pydantic validators enforce IPv6 address correctness in request bodies. Rate-limit by /64 subnets to handle the large address space IPv6 clients may use. Monitor FastAPI with OneUptime's API checks targeting IPv6 endpoints.
