# How to Use count.index for Sequential Naming in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, HCL, Count, Naming, Resources, Infrastructure as Code

Description: Learn how to use count.index in OpenTofu to create sequentially named resources like subnets, instances, and security groups.

---

The `count` meta-argument in OpenTofu creates multiple instances of a resource. The `count.index` value (0-based) lets you generate sequential names, assign availability zones, and calculate CIDR blocks dynamically.

---

## Basic Sequential Naming

```hcl
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)

  tags = {
    Name = "public-subnet-${count.index + 1}"  # 1-based naming
  }
}
# Creates: public-subnet-1, public-subnet-2, public-subnet-3

```

---

## Using a Name Prefix Variable

```hcl
variable "name_prefix" {
  default = "web"
}

resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  tags = {
    Name  = "${var.name_prefix}-${count.index + 1}"
    Index = tostring(count.index)
  }
}
# Creates: web-1, web-2, web-3
```

---

## Assign Availability Zones with count.index

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-subnet-${data.aws_availability_zones.available.names[count.index]}"
  }
}
```

---

## Reference count Resources

```hcl
# Reference all instances as a list
output "web_private_ips" {
  value = aws_instance.web[*].private_ip
}

# Reference a specific instance
output "first_web_ip" {
  value = aws_instance.web[0].private_ip
}
```

---

## count vs for_each

| Use Case                          | Prefer    |
|-----------------------------------|-----------|
| Identical resources, just N copies| `count`   |
| Resources with distinct attributes| `for_each`|
| Sequential numeric naming         | `count`   |
| Named resources (e.g., `us-east-1`)| `for_each`|

---

## Formatting Sequential Names

```hcl
# Zero-padded names: node-001, node-002, node-010
name = format("node-%03d", count.index + 1)
```

---

## Summary

Use `count = N` to create multiple identical resources and `count.index` (0-based) to generate sequential names. Add `1` to get 1-based naming. Use `count.index` to index into lists like availability zone names or CIDR offsets. Reference all instances with the splat operator `[*]` or individual instances with `[N]`.
