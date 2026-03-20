# How OpenTofu Represents Infrastructure Changes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Plan Output, Change Representation, Infrastructure as Code, DevOps

Description: Learn how to read OpenTofu plan output - understanding the symbols, colors, and JSON change representations that indicate what will be created, updated, replaced, or destroyed.

## Introduction

OpenTofu's plan output uses a consistent set of symbols and annotations to represent what will happen to each resource. Reading plans accurately is a critical skill - it's your last line of defense before changes reach production.

## Plan Symbols

```text
+ create          Green:  New resource will be created
- destroy         Red:    Resource will be deleted
~ update in-place Yellow: Resource modified without recreation
+/- create then destroy: New resource created before old is deleted (create_before_destroy)
-/+ destroy then create: Old resource deleted before new is created (requires recreation)
<= read           Cyan:   Data source will be read
(known after apply)        Attribute value not known until apply
```

## Create (+)

```hcl
# This resource doesn't exist in state - will be created

resource "aws_s3_bucket" "new" {
  bucket = "my-new-bucket"
}
```

Plan output:

```hcl
  # aws_s3_bucket.new will be created
  + resource "aws_s3_bucket" "new" {
      + arn                         = (known after apply)
      + bucket                      = "my-new-bucket"
      + bucket_domain_name          = (known after apply)
      + id                          = (known after apply)
    }
```

## Update In-Place (~)

```hcl
# Only the tags changed - AWS can update this without recreation
resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
  tags = { Environment = "prod" }  # was "staging"
}
```

Plan output:

```hcl
  # aws_s3_bucket.existing will be updated in-place
  ~ resource "aws_s3_bucket" "existing" {
      ~ tags = {
          ~ "Environment" = "staging" -> "prod"
        }
        id     = "my-existing-bucket"
    }
```

## Destroy and Recreate (-/+)

When you change an immutable attribute, the resource must be destroyed and recreated:

```hcl
# Changing ami requires instance recreation
resource "aws_instance" "web" {
  ami           = "ami-new-version"  # changed from old AMI
  instance_type = "t3.medium"
}
```

Plan output:

```hcl
  # aws_instance.web must be replaced
-/+ resource "aws_instance" "web" {
      ~ ami                   = "ami-old-version" -> "ami-new-version" # forces replacement
        instance_type         = "t3.medium"
      + id                    = (known after apply)
    }
```

The `# forces replacement` annotation identifies which attribute change is causing recreation.

## Create Before Destroy (+/-)

With `create_before_destroy = true` in the lifecycle block:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-new-version"
  instance_type = "t3.medium"

  lifecycle {
    create_before_destroy = true
  }
}
```

Plan output shows `+/-` instead of `-/+`:

```hcl
  # aws_instance.web must be replaced
+/- resource "aws_instance" "web" {
      ~ ami = "ami-old-version" -> "ami-new-version" # forces replacement
    }
```

## JSON Representation of Changes

For programmatic processing:

```bash
tofu plan -out=tfplan.binary
tofu show -json tfplan.binary | jq '.resource_changes[] | {
  address: .address,
  actions: .change.actions,
  before: .change.before,
  after: .change.after,
  after_unknown: .change.after_unknown
}'
```

Actions array in JSON:

```json
{ "actions": ["create"] }          // +
{ "actions": ["delete"] }          // -
{ "actions": ["update"] }          // ~
{ "actions": ["delete", "create"] } // -/+
{ "actions": ["create", "delete"] } // +/-
{ "actions": ["no-op"] }           // no changes
```

## Reading Sensitive Value Changes

```hcl
  # aws_db_instance.postgres will be updated in-place
  ~ resource "aws_db_instance" "postgres" {
      ~ password = (sensitive value)
        id       = "prod-postgres"
    }
```

Sensitive values show as `(sensitive value)` - they're changed but not displayed in the plan.

## Reading (known after apply)

```hcl
  + resource "aws_instance" "web" {
      + id         = (known after apply)
      + public_ip  = (known after apply)
      + private_ip = (known after apply)
      + ami        = "ami-12345678"
    }
```

`(known after apply)` means the value is computed by the provider during creation. Any resource that depends on these values will also show `(known after apply)` for those attributes.

## Red Flags in Plan Output

Watch for these before approving:

```hcl
1. Unexpected -/+ (recreation): Will cause downtime
   Check if there's a lifecycle.create_before_destroy option

2. Unexpected - (destroy): Check if this is intentional
   Run: tofu state list | grep resource-name to verify it exists

3. Large numbers of changes: Something might be wrong
   Check for provider version updates that changed defaults

4. ~ tags = {} -> null: Tags are being removed unexpectedly
   Check if provider default_tags changed
```

## Conclusion

Reading OpenTofu plan output accurately requires understanding the symbols (`+`, `-`, `~`, `-/+`, `+/-`), the distinction between update-in-place and recreation, and what `(known after apply)` means for dependent resources. The JSON plan format (`tofu show -json`) exposes the same information programmatically, enabling automated plan review scripts to catch unexpected destroys and recreations before they reach production.
