# How to Validate IPv6 Addresses in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Python, IPv6, Validation, Ipaddress, Input Validation, Programming

Description: Validate IPv6 addresses in Python using the ipaddress module, custom validators, and form validation libraries for web applications.

## Basic Validation with ipaddress

The most reliable way to validate IPv6 in Python:

```python
import ipaddress

def is_valid_ipv6(address: str) -> bool:
    """Return True if address is a valid IPv6 address."""
    try:
        ipaddress.IPv6Address(address)
        return True
    except ValueError:
        return False

# Tests

test_cases = [
    ("2001:db8::1", True),
    ("::1", True),
    ("fe80::1", True),
    ("::", True),
    ("2001:db8:0:0:0:0:0:1", True),
    ("2001:db8::xyz", False),      # Invalid hex
    ("2001:db8:::1", False),       # Triple colon
    ("192.168.1.1", False),        # IPv4 address
    ("", False),
    ("GGGG::1", False),            # Invalid hex digit
]

for addr, expected in test_cases:
    result = is_valid_ipv6(addr)
    status = "OK" if result == expected else "FAIL"
    print(f"[{status}] '{addr}' → {result}")
```

## Validating IPv6 with Prefix Length (CIDR)

```python
import ipaddress

def is_valid_ipv6_cidr(cidr: str) -> bool:
    """Validate an IPv6 CIDR notation (address/prefix-length)."""
    try:
        ipaddress.IPv6Network(cidr, strict=False)
        return True
    except ValueError:
        return False

# Tests
cidr_tests = [
    "2001:db8::/32",       # Valid network
    "2001:db8::1/128",     # Valid host route
    "::/0",                # Default route
    "2001:db8::/129",      # Invalid (prefix > 128)
    "2001:xyz::/48",       # Invalid address
]

for cidr in cidr_tests:
    print(f"{cidr:25} → {is_valid_ipv6_cidr(cidr)}")
```

## Typed Validation with Type Annotations

Create a reusable validator type for use in dataclasses and Pydantic:

```python
from ipaddress import IPv6Address
from typing import Annotated

# Using Pydantic v2 for robust validation
from pydantic import BaseModel, field_validator, ValidationError

class NetworkDevice(BaseModel):
    name: str
    ipv6_address: str
    ipv6_prefix: str

    @field_validator("ipv6_address")
    @classmethod
    def validate_ipv6(cls, v: str) -> str:
        try:
            IPv6Address(v)
        except ValueError as e:
            raise ValueError(f"Invalid IPv6 address: {v}") from e
        return v

    @field_validator("ipv6_prefix")
    @classmethod
    def validate_prefix(cls, v: str) -> str:
        import ipaddress
        try:
            ipaddress.IPv6Network(v, strict=True)
        except ValueError as e:
            raise ValueError(f"Invalid IPv6 prefix: {v}") from e
        return v

# Test
try:
    device = NetworkDevice(
        name="router-1",
        ipv6_address="2001:db8::1",
        ipv6_prefix="2001:db8::/32"
    )
    print(f"Valid: {device}")
except ValidationError as e:
    print(f"Validation error: {e}")
```

## Validating Specific Address Types

Sometimes you need to validate not just the format but the type of address:

```python
import ipaddress

def validate_global_unicast(address: str) -> tuple[bool, str]:
    """
    Validate that address is a valid global unicast IPv6 address.
    Returns (is_valid, reason).
    """
    try:
        addr = ipaddress.IPv6Address(address)
    except ValueError:
        return False, "Not a valid IPv6 address"

    if addr.is_loopback:
        return False, "Loopback address not allowed"
    if addr.is_link_local:
        return False, "Link-local address not allowed"
    if addr.is_private:
        return False, "ULA (private) address not allowed"
    if addr.is_multicast:
        return False, "Multicast address not allowed"
    if addr.is_unspecified:
        return False, "Unspecified address not allowed"
    if not addr.is_global:
        return False, "Not a global unicast address"

    return True, "Valid global unicast address"

# Tests
for addr in ["2001:db8::1", "::1", "fe80::1", "fd00::1", "ff02::1"]:
    valid, reason = validate_global_unicast(addr)
    print(f"{addr:25} → {valid} ({reason})")
```

## Web Form Validation (Flask)

Validate IPv6 input in a Flask web application:

```python
from flask import Flask, request, jsonify
import ipaddress

app = Flask(__name__)

@app.route("/api/device", methods=["POST"])
def add_device():
    data = request.get_json()
    ipv6_addr = data.get("ipv6_address", "")

    # Validate
    try:
        addr = ipaddress.IPv6Address(ipv6_addr)
    except ValueError:
        return jsonify({"error": f"Invalid IPv6 address: {ipv6_addr}"}), 400

    # Continue processing with valid address
    return jsonify({"status": "ok", "address": str(addr.compressed)})
```

## Using wtforms for Form Validation

```python
from wtforms import Form, StringField, validators
import ipaddress

class DeviceForm(Form):
    ipv6_address = StringField("IPv6 Address", [
        validators.DataRequired(),
        validators.Length(min=2, max=45)  # Max IPv6 length is 45 chars
    ])

    def validate_ipv6_address(self, field):
        try:
            ipaddress.IPv6Address(field.data)
        except ValueError:
            raise validators.ValidationError("Invalid IPv6 address format")
```

## Conclusion

Python's `ipaddress.IPv6Address` constructor provides the most reliable IPv6 validation - it handles all valid formats including compressed notation, loopback, and link-local addresses. For web applications, wrap the validation in Pydantic validators or wtforms validators for clean integration with your request handling code.
