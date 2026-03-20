# How to Validate IPv6 Addresses in Go

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, IPv6, Validation, net/netip, Input Validation, Programming

Description: Validate IPv6 addresses in Go using net/netip, with custom validators for specific address types and HTTP handler integration.

## Basic Validation

```go
package main

import (
    "fmt"
    "net/netip"
)

// IsValidIPv6 returns true if the string is a valid IPv6 address.
func IsValidIPv6(s string) bool {
    addr, err := netip.ParseAddr(s)
    if err != nil {
        return false
    }
    return addr.Is6() || addr.Is4In6()
}

func main() {
    tests := []string{
        "2001:db8::1",     // valid
        "::1",             // valid (loopback)
        "fe80::1",         // valid (link-local)
        "::",              // valid (unspecified)
        "192.168.1.1",     // invalid (IPv4)
        "2001:xyz::1",     // invalid (bad hex)
        "",                // invalid
        "::gggg",          // invalid
    }

    for _, t := range tests {
        fmt.Printf("%-25s → valid=%v\n", t, IsValidIPv6(t))
    }
}
```

## Validating Specific Address Types

```go
package main

import (
    "errors"
    "fmt"
    "net/netip"
)

var (
    ErrNotIPv6         = errors.New("not a valid IPv6 address")
    ErrLinkLocal       = errors.New("link-local address not allowed")
    ErrLoopback        = errors.New("loopback address not allowed")
    ErrPrivate         = errors.New("private (ULA) address not allowed")
    ErrMulticast       = errors.New("multicast address not allowed")
    ErrNotGlobalUnicast = errors.New("must be a global unicast address")
)

// ValidateGlobalUnicast validates that an address is a global unicast IPv6.
func ValidateGlobalUnicast(s string) error {
    addr, err := netip.ParseAddr(s)
    if err != nil || !addr.Is6() {
        return ErrNotIPv6
    }
    if addr.IsLoopback() {
        return ErrLoopback
    }
    if addr.IsLinkLocalUnicast() {
        return ErrLinkLocal
    }
    if addr.IsPrivate() {
        return ErrPrivate
    }
    if addr.IsMulticast() {
        return ErrMulticast
    }
    if !addr.IsGlobalUnicast() {
        return ErrNotGlobalUnicast
    }
    return nil
}

// ValidateIPv6CIDR validates an IPv6 network prefix in CIDR notation.
func ValidateIPv6CIDR(s string) error {
    prefix, err := netip.ParsePrefix(s)
    if err != nil {
        return fmt.Errorf("invalid CIDR: %w", err)
    }
    if !prefix.Addr().Is6() {
        return ErrNotIPv6
    }
    return nil
}
```

## HTTP Handler with IPv6 Validation

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "net/netip"
)

type DeviceRequest struct {
    Name       string `json:"name"`
    IPv6Address string `json:"ipv6_address"`
}

type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

func validateDeviceRequest(req DeviceRequest) []ValidationError {
    var errors []ValidationError

    if req.Name == "" {
        errors = append(errors, ValidationError{"name", "required"})
    }

    addr, err := netip.ParseAddr(req.IPv6Address)
    if err != nil || !addr.Is6() {
        errors = append(errors, ValidationError{
            "ipv6_address",
            fmt.Sprintf("invalid IPv6 address: %s", req.IPv6Address),
        })
    }

    return errors
}

func createDeviceHandler(w http.ResponseWriter, r *http.Request) {
    var req DeviceRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid JSON", http.StatusBadRequest)
        return
    }

    validationErrors := validateDeviceRequest(req)
    if len(validationErrors) > 0 {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusUnprocessableEntity)
        json.NewEncoder(w).Encode(map[string]interface{}{
            "errors": validationErrors,
        })
        return
    }

    // Process valid request
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{"status": "created"})
}
```

## Custom Validator with go-validator

```go
package main

import (
    "fmt"
    "net/netip"

    "github.com/go-playground/validator/v10"
)

// Register a custom IPv6 validator for go-playground/validator
func registerIPv6Validator(validate *validator.Validate) {
    validate.RegisterValidation("ipv6addr", func(fl validator.FieldLevel) bool {
        s := fl.Field().String()
        addr, err := netip.ParseAddr(s)
        return err == nil && addr.Is6()
    })
}

type Device struct {
    Name    string `validate:"required,min=1,max=100"`
    Address string `validate:"required,ipv6addr"`
}

func main() {
    validate := validator.New()
    registerIPv6Validator(validate)

    device := Device{
        Name:    "router-1",
        Address: "2001:db8::1",
    }

    if err := validate.Struct(device); err != nil {
        fmt.Println("Validation failed:", err)
        return
    }
    fmt.Println("Device is valid")
}
```

## Table-Driven Validation Tests

```go
package main

import (
    "testing"
)

func TestIsValidIPv6(t *testing.T) {
    tests := []struct {
        input    string
        expected bool
    }{
        {"2001:db8::1", true},
        {"::1", true},
        {"fe80::1", true},
        {"::", true},
        {"192.168.1.1", false},
        {"not-an-ip", false},
        {"", false},
        {"2001:db8:::1", false},
    }

    for _, tt := range tests {
        t.Run(tt.input, func(t *testing.T) {
            result := IsValidIPv6(tt.input)
            if result != tt.expected {
                t.Errorf("IsValidIPv6(%q) = %v, want %v",
                    tt.input, result, tt.expected)
            }
        })
    }
}
```

## Conclusion

Go's `net/netip.ParseAddr()` is the most reliable way to validate IPv6 addresses. It handles all valid forms including compressed notation, link-local with zone IDs, and IPv4-mapped addresses. For web applications, use the `go-playground/validator` library with a custom IPv6 validation function registered via `RegisterValidation`.
