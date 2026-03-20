# How to Use Go for IPv6 Network Automation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Go, IPv6, Network Automation, SSH, REST API, NETCONF

Description: Use Go for IPv6 network automation tasks including SSH-based configuration, REST API interactions, and NETCONF device management.

## SSH-Based Device Configuration

Automate IPv6 configuration on network devices via SSH using the `golang.org/x/crypto/ssh` package:

```go
package main

import (
    "fmt"
    "strings"
    "time"

    "golang.org/x/crypto/ssh"
)

type NetworkDevice struct {
    Host     string  // IPv6 address of the device
    Port     int
    Username string
    Password string
}

func (d *NetworkDevice) Connect() (*ssh.Client, error) {
    config := &ssh.ClientConfig{
        User: d.Username,
        Auth: []ssh.AuthMethod{
            ssh.Password(d.Password),
        },
        HostKeyCallback: ssh.InsecureIgnoreHostKey(), // Use proper verification in production
        Timeout:         10 * time.Second,
    }

    addr := fmt.Sprintf("[%s]:%d", d.Host, d.Port)
    return ssh.Dial("tcp", addr, config)
}

func (d *NetworkDevice) RunCommand(client *ssh.Client, cmd string) (string, error) {
    session, err := client.NewSession()
    if err != nil {
        return "", err
    }
    defer session.Close()

    var stdout strings.Builder
    session.Stdout = &stdout

    if err := session.Run(cmd); err != nil {
        return "", err
    }
    return stdout.String(), nil
}

func configureIPv6Route(device *NetworkDevice, prefix, nextHop string) error {
    client, err := device.Connect()
    if err != nil {
        return fmt.Errorf("SSH connect to [%s]: %w", device.Host, err)
    }
    defer client.Close()

    // Configure a static IPv6 route (IOS syntax)
    cmd := fmt.Sprintf(
        "configure terminal\nipv6 route %s %s\nend\nwrite memory\n",
        prefix, nextHop,
    )

    output, err := device.RunCommand(client, cmd)
    if err != nil {
        return fmt.Errorf("command failed: %w", err)
    }

    fmt.Printf("Device [%s]: %s\n", device.Host, output)
    return nil
}

func main() {
    router := &NetworkDevice{
        Host:     "2001:db8::router1",
        Port:     22,
        Username: "admin",
        Password: "secret",
    }

    err := configureIPv6Route(router, "2001:db8:remote::/48", "2001:db8::gateway")
    if err != nil {
        fmt.Println("Error:", err)
    }
}
```

## REST API Automation (NetBox IPAM)

Automate IPv6 prefix allocation in NetBox via its REST API:

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type NetBoxClient struct {
    BaseURL string
    Token   string
    HTTP    *http.Client
}

type PrefixRequest struct {
    Prefix      string `json:"prefix"`
    Status      string `json:"status"`
    Description string `json:"description"`
    IsPool      bool   `json:"is_pool"`
}

func (c *NetBoxClient) CreateIPv6Prefix(prefix, description string) error {
    payload := PrefixRequest{
        Prefix:      prefix,
        Status:      "active",
        Description: description,
        IsPool:      false,
    }

    body, err := json.Marshal(payload)
    if err != nil {
        return err
    }

    req, err := http.NewRequest(
        "POST",
        c.BaseURL+"/api/ipam/prefixes/",
        bytes.NewBuffer(body),
    )
    if err != nil {
        return err
    }

    req.Header.Set("Authorization", "Token "+c.Token)
    req.Header.Set("Content-Type", "application/json")

    resp, err := c.HTTP.Do(req)
    if err != nil {
        return fmt.Errorf("API request failed: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusCreated {
        return fmt.Errorf("API error: %d", resp.StatusCode)
    }

    fmt.Printf("Created IPv6 prefix: %s\n", prefix)
    return nil
}

func main() {
    client := &NetBoxClient{
        BaseURL: "http://[2001:db8::netbox]",
        Token:   "your-api-token",
        HTTP:    &http.Client{},
    }

    prefixes := []struct{ prefix, desc string }{
        {"2001:db8:1::/48", "Customer A Production"},
        {"2001:db8:2::/48", "Customer B Production"},
        {"2001:db8:3::/48", "Customer C Staging"},
    }

    for _, p := range prefixes {
        if err := client.CreateIPv6Prefix(p.prefix, p.desc); err != nil {
            fmt.Printf("Error creating %s: %v\n", p.prefix, err)
        }
    }
}
```

## Batch Address Validation and IPAM Report

```go
package main

import (
    "encoding/csv"
    "fmt"
    "net/netip"
    "os"
)

type IPAMRecord struct {
    Name    string
    Address string
    Valid   bool
    Type    string
}

func validateAndReport(records []IPAMRecord) {
    w := csv.NewWriter(os.Stdout)
    w.Write([]string{"Name", "Address", "Valid", "Type", "Compressed"})

    for _, r := range records {
        addr, err := netip.ParseAddr(r.Address)
        valid := err == nil && addr.Is6()

        addrType := "invalid"
        compressed := r.Address
        if valid {
            compressed = addr.String()
            switch {
            case addr.IsLoopback():
                addrType = "loopback"
            case addr.IsLinkLocalUnicast():
                addrType = "link-local"
            case addr.IsPrivate():
                addrType = "ULA"
            case addr.IsGlobalUnicast():
                addrType = "global-unicast"
            case addr.IsMulticast():
                addrType = "multicast"
            }
        }

        w.Write([]string{
            r.Name,
            r.Address,
            fmt.Sprintf("%v", valid),
            addrType,
            compressed,
        })
    }
    w.Flush()
}
```

## Conclusion

Go is an excellent language for IPv6 network automation due to its standard library support for SSH, HTTP, and low-level networking. Combined with libraries like `golang.org/x/crypto/ssh` for device access and standard `net/http` for REST API integration, Go enables building robust automation tools that scale from single-device configuration to fleet-wide IPv6 deployment management.
