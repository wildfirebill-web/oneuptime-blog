# How to Use the csvdecode Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the csvdecode function in OpenTofu to parse CSV data into a list of maps for bulk resource creation from spreadsheet-like inputs.

## Introduction

The `csvdecode` function in OpenTofu parses a CSV string into a list of maps, where the first row provides the column names (keys). This is useful for bulk resource creation from CSV configuration files, importing data from spreadsheets, and managing large inventories.

## Syntax

```hcl
csvdecode(string)
```

- The first row must be a header row defining the keys
- Returns a `list(map(string))`

## Basic Examples

```hcl
output "parsed_csv" {
  value = csvdecode("name,size\nsmall,10\nmedium,50\nlarge,100")
  # Returns:
  # [
  #   { name = "small",  size = "10"  },
  #   { name = "medium", size = "50"  },
  #   { name = "large",  size = "100" }
  # ]
}
```

## Practical Use Cases

### Creating EC2 Instances from CSV

```hcl
# instances.csv:

# name,instance_type,environment
# web-1,t3.micro,dev
# web-2,t3.medium,staging
# api-1,m5.large,prod

locals {
  instances_csv = file("${path.module}/data/instances.csv")
  instances     = csvdecode(local.instances_csv)
}

resource "aws_instance" "servers" {
  for_each      = { for inst in local.instances : inst.name => inst }
  ami           = data.aws_ami.ubuntu.id
  instance_type = each.value.instance_type

  tags = {
    Name        = each.key
    Environment = each.value.environment
  }
}
```

### Bulk DNS Record Creation

```hcl
# dns_records.csv:
# subdomain,type,value
# api,A,1.2.3.4
# www,CNAME,api.example.com
# mail,MX,mail.example.com

locals {
  dns_csv     = file("${path.module}/data/dns.csv")
  dns_records = csvdecode(local.dns_csv)
}

resource "aws_route53_record" "records" {
  for_each = { for r in local.dns_records : r.subdomain => r }

  zone_id = aws_route53_zone.main.zone_id
  name    = "${each.key}.example.com"
  type    = each.value.type
  ttl     = 300
  records = [each.value.value]
}
```

### IAM Users from CSV

```hcl
# users.csv:
# username,email,team
# alice,alice@example.com,platform
# bob,bob@example.com,data

locals {
  users_csv = file("${path.module}/data/users.csv")
  users     = csvdecode(local.users_csv)
}

resource "aws_iam_user" "team" {
  for_each = { for u in local.users : u.username => u }
  name     = each.key

  tags = {
    Email = each.value.email
    Team  = each.value.team
  }
}
```

### Inline CSV Data

```hcl
locals {
  subnets = csvdecode(<<-CSV
    az,cidr,public
    us-east-1a,10.0.1.0/24,true
    us-east-1b,10.0.2.0/24,true
    us-east-1c,10.0.3.0/24,false
  CSV
  )
}

resource "aws_subnet" "vpc" {
  for_each = { for s in local.subnets : s.az => s }

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.key
  map_public_ip_on_launch = each.value.public == "true"

  tags = {
    Name = "subnet-${each.key}"
  }
}
```

## Step-by-Step Usage

1. Prepare a CSV string with a header row.
2. Call `csvdecode(csv_string)` to get a list of maps.
3. Use `for` expressions to convert to a keyed map for `for_each`.
4. Test in `tofu console`:

```bash
tofu console

> csvdecode("name,age\nalice,30\nbob,25")
[{age = "30", name = "alice"}, {age = "25", name = "bob"}]
```

## Important Notes

- All values are strings - convert numbers with `tonumber()` as needed.
- CSV values with commas must be quoted: `"value,with,commas"`.
- Empty cells produce empty strings.

## Conclusion

The `csvdecode` function enables data-driven infrastructure in OpenTofu by parsing CSV files into structured data. Combined with `file()` and `for_each`, it allows you to manage large inventories of resources from spreadsheet-like inputs, making your infrastructure truly configuration-driven.
