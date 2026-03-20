# How to Configure Django for IPv6 Support

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Django, Python, IPv6, ALLOWED_HOSTS, Web Framework, WSGI, ASGI

Description: Configure Django to accept requests from IPv6 clients by updating ALLOWED_HOSTS, handling IPv6 in middleware, and deploying with an IPv6-capable application server.

## Introduction

Django requires explicit configuration to work with IPv6. The key settings are `ALLOWED_HOSTS` (which must include IPv6 addresses in bracket notation for the dev server), and the `REMOTE_ADDR` handling in middleware. Production deployments need Gunicorn or uWSGI bound to IPv6.

## Step 1: ALLOWED_HOSTS for IPv6

```python
# settings.py

# Allow IPv6 addresses and hostnames

ALLOWED_HOSTS = [
    "example.com",
    "www.example.com",
    "2001:db8::1",          # Specific IPv6 address
    "[2001:db8::1]",        # Bracket notation (for dev server)
    "::1",                  # Loopback
    "[::1]",
    # Wildcard for all (dev only!)
    # "*",
]

# For the development server on IPv6:
# python manage.py runserver [::]:8000
# Requires the bracket notation in ALLOWED_HOSTS
```

## Step 2: Run Django Dev Server on IPv6

```bash
# Listen on all IPv6 interfaces
python manage.py runserver "[::]:8000"

# Listen on loopback IPv6 only
python manage.py runserver "[::1]:8000"

# Listen on specific IPv6 address
python manage.py runserver "[2001:db8::1]:8000"
```

## Step 3: Middleware for IPv6 Client IP

```python
# myapp/middleware.py

import ipaddress
from django.http import HttpRequest

class IPv6ClientMiddleware:
    """
    Normalize IPv6 client addresses and handle X-Forwarded-For.
    """
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request: HttpRequest):
        # Extract real client IP
        xff = request.META.get("HTTP_X_FORWARDED_FOR", "")
        if xff:
            ip = xff.split(",")[0].strip().strip("[]")
        else:
            ip = request.META.get("REMOTE_ADDR", "")

        # Normalize IPv6
        try:
            addr = ipaddress.ip_address(ip)
            request.client_ip = str(addr)
            request.client_ip_version = addr.version
        except ValueError:
            request.client_ip = ip
            request.client_ip_version = None

        return self.get_response(request)
```

```python
# settings.py
MIDDLEWARE = [
    "myapp.middleware.IPv6ClientMiddleware",
    "django.middleware.security.SecurityMiddleware",
    # ...
]
```

## Step 4: Store and Display IPv6 in Models

```python
# models.py
from django.db import models
from django.core.validators import validate_ipv46_address

class UserSession(models.Model):
    # GenericIPAddressField handles both IPv4 and IPv6
    ip_address = models.GenericIPAddressField(
        protocol="both",       # Accept IPv4 and IPv6
        unpack_ipv4=True,      # Unpack ::ffff:1.2.3.4 → 1.2.3.4
    )
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"Session from {self.ip_address}"
```

## Step 5: Forms with IPv6 Validation

```python
# forms.py
from django import forms
from django.core.validators import validate_ipv46_address

class NetworkConfigForm(forms.Form):
    ipv6_address = forms.GenericIPAddressField(
        protocol="IPv6",
        label="IPv6 Address",
    )

    def clean_ipv6_address(self):
        import ipaddress
        addr_str = self.cleaned_data["ipv6_address"]
        try:
            addr = ipaddress.IPv6Address(addr_str)
            if addr.is_loopback and not self.allow_loopback:
                raise forms.ValidationError("Loopback addresses not permitted")
            return str(addr)
        except ValueError:
            raise forms.ValidationError("Invalid IPv6 address")
```

## Step 6: Production with Gunicorn/ASGI

```bash
# Gunicorn WSGI
gunicorn myproject.wsgi:application \
    --bind "[::]:8000" \
    --workers 4

# Uvicorn ASGI (for Django Channels or async views)
uvicorn myproject.asgi:application \
    --host "::" \
    --port 8000 \
    --workers 4
```

## Step 7: CSRF and Session Cookies on IPv6

```python
# settings.py

# Ensure session cookies work for IPv6 clients
SESSION_COOKIE_DOMAIN = None  # Use the request's host

# CSRF trusted origins - include IPv6 addresses
CSRF_TRUSTED_ORIGINS = [
    "https://example.com",
    "https://[2001:db8::1]",
    "http://[::1]:8000",
]
```

## Conclusion

Django IPv6 support requires updating `ALLOWED_HOSTS` with IPv6 addresses (in bracket notation for the dev server), using `GenericIPAddressField` for storing IPs in models, and binding Gunicorn/Uvicorn to `[::]:8000`. Add `ProxyFix` or custom middleware to extract real IPv6 client addresses from proxy headers. Monitor Django with OneUptime's uptime and SSL checks on IPv6 endpoints.
