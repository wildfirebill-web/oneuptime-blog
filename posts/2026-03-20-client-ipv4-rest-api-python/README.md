# How to Get the Client IPv4 Address from REST API Requests in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, Flask, FastAPI, IPv4, REST API, Networking

Description: Learn how to extract the real client IPv4 address from REST API requests in Python using Flask and FastAPI, handling reverse proxy X-Forwarded-For headers and direct connections correctly.

## Flask: Direct Connection

```python
from flask import Flask, request

app = Flask(__name__)

@app.route("/whoami")
def whoami():
    # request.remote_addr is correct when there is no reverse proxy
    return {"client_ip": request.remote_addr}
```

## Flask: Behind a Reverse Proxy (Nginx / AWS ALB)

```python
from flask import Flask, request

app = Flask(__name__)

# Tell Flask to trust the first proxy in the chain

app.config["TRUSTED_PROXIES"] = 1
# Flask 2.3+ uses ProxyFix middleware
from werkzeug.middleware.proxy_fix import ProxyFix
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1)

@app.route("/whoami")
def whoami():
    # After ProxyFix, request.remote_addr reflects the real client IP
    return {"client_ip": request.remote_addr}
```

## Flask: Manual X-Forwarded-For Parsing

```python
from flask import Flask, request
import ipaddress

app = Flask(__name__)

TRUSTED_PROXIES = [
    ipaddress.IPv4Network("10.0.0.0/8"),
    ipaddress.IPv4Network("172.16.0.0/12"),
    ipaddress.IPv4Network("192.168.0.0/16"),
    ipaddress.IPv4Network("127.0.0.0/8"),
]

def is_trusted_proxy(ip: str) -> bool:
    try:
        addr = ipaddress.IPv4Address(ip)
        return any(addr in net for net in TRUSTED_PROXIES)
    except ValueError:
        return False

def get_real_ip() -> str:
    xff = request.headers.get("X-Forwarded-For", "")
    if xff and is_trusted_proxy(request.remote_addr):
        # Take the leftmost IP - the original client
        return xff.split(",")[0].strip()
    return request.remote_addr

@app.route("/whoami")
def whoami():
    return {"client_ip": get_real_ip()}
```

## FastAPI

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/whoami")
async def whoami(request: Request):
    # request.client.host is the direct connection IP
    xff = request.headers.get("x-forwarded-for")
    if xff:
        client_ip = xff.split(",")[0].strip()
    else:
        client_ip = request.client.host
    return {"client_ip": client_ip}
```

## FastAPI with TrustedHostMiddleware Proxy Support

```python
from fastapi import FastAPI, Request
from starlette.middleware.trustedhost import TrustedHostMiddleware

app = FastAPI()

# When deployed behind a known proxy, read forwarded headers safely
@app.middleware("http")
async def extract_real_ip(request: Request, call_next):
    xff = request.headers.get("x-forwarded-for")
    if xff:
        request.state.client_ip = xff.split(",")[0].strip()
    else:
        request.state.client_ip = request.client.host
    return await call_next(request)

@app.get("/whoami")
async def whoami(request: Request):
    return {"client_ip": request.state.client_ip}
```

## Conclusion

Never read `X-Forwarded-For` without first verifying that `request.remote_addr` is a trusted proxy - an attacker can spoof this header on direct connections. In Flask, use `werkzeug.middleware.proxy_fix.ProxyFix` with `x_for=1` to safely unwrap one level of proxy. In FastAPI, validate the incoming connection address before trusting forwarded headers. Always document the expected number of proxy hops so the configuration remains correct as infrastructure changes.
