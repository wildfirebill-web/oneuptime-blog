# How to Configure Docker Daemon Default Address Pools for IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Docker, Networking, IPv4, daemon.json, Address Pool, Configuration

Description: Configure Docker daemon default address pools in /etc/docker/daemon.json to control which IPv4 subnets are used when creating new bridge networks without an explicit subnet.

## Introduction

When you run `docker network create` without specifying a subnet, Docker picks from its default address pool. Configuring `default-address-pools` in `daemon.json` ensures Docker chooses subnets from a range you control, preventing conflicts with your existing network infrastructure.

## Configuring daemon.json

```bash
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "default-address-pools": [
    {
      "base": "10.200.0.0/16",
      "size": 24
    },
    {
      "base": "10.201.0.0/16",
      "size": 24
    }
  ]
}
EOF
```

- `base`: the parent range from which Docker will carve subnets
- `size`: the prefix length of each allocated subnet (24 = /24)

With this config, when you run `docker network create mynet`, Docker will pick `10.200.0.0/24`, then `10.200.1.0/24`, and so on.

## Applying the Configuration

```bash
sudo systemctl restart docker

# Verify by creating a network without specifying a subnet

docker network create test-pool
docker network inspect test-pool | grep '"Subnet"'
# Should show: "Subnet": "10.200.0.0/24"
```

## Combining bip and address pools

```json
{
  "bip": "192.168.90.1/24",
  "default-address-pools": [
    {
      "base": "10.200.0.0/16",
      "size": 24
    }
  ]
}
```

- `bip`: controls the `docker0` (default bridge) subnet
- `default-address-pools`: controls all new user-defined bridge networks

## Multiple Pools for Different Purposes

```json
{
  "default-address-pools": [
    {
      "base": "10.200.0.0/16",
      "size": 24
    },
    {
      "base": "10.201.0.0/16",
      "size": 28
    }
  ]
}
```

Docker exhausts the first pool before moving to the second. Use `/28` subnets for tiny networks (14 hosts max) to conserve address space.

## Viewing Current Address Pool Allocation

```bash
# See all networks and their subnets
docker network ls --format '{{.Name}}' | xargs -I {} docker network inspect {} --format '{{.Name}}: {{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

## Planning Address Pool Sizes

| Use Case | Recommended base | size |
|---|---|---|
| General purpose | 10.200.0.0/16 | 24 |
| Microservices with many networks | 10.200.0.0/14 | 24 |
| Small test networks | 10.201.0.0/16 | 28 |

## Conclusion

Configure `default-address-pools` in `/etc/docker/daemon.json` to direct Docker to your preferred IPv4 ranges when creating bridge networks without explicit subnets. This is especially important in environments where `172.16.0.0/12` or `192.168.0.0/16` ranges conflict with VPN or corporate infrastructure.
