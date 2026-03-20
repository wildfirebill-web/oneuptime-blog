# Using tofu state mv in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, State

Description: Learn how to use tofu state mv to rename resources, move them between modules, and refactor your infrastructure configuration without recreation.

The `tofu state mv` command moves resources within the state file - renaming them, reorganizing module structure, or migrating between configurations - without destroying and recreating the actual infrastructure.

## Basic Syntax

```bash
tofu state mv [options] SOURCE DESTINATION
```

## Renaming a Resource

```bash
# Rename aws_instance.server to aws_instance.web_server

tofu state mv aws_instance.server aws_instance.web_server

# Output:
# Move "aws_instance.server" to "aws_instance.web_server"
# Successfully moved 1 object(s).
```

Then update your configuration:

```hcl
# Before
resource "aws_instance" "server" { ... }

# After
resource "aws_instance" "web_server" { ... }
```

## Moving a Resource into a Module

```bash
# Move resource from root to a module
tofu state mv aws_security_group.web module.application.aws_security_group.web

# Move multiple resources
tofu state mv aws_vpc.main module.networking.aws_vpc.main
tofu state mv aws_subnet.public module.networking.aws_subnet.public
tofu state mv aws_subnet.private module.networking.aws_subnet.private
```

## Moving a Resource out of a Module

```bash
# Move from module to root
tofu state mv module.networking.aws_eip.nat aws_eip.nat
```

## Moving Between Modules

```bash
# Move resource from one module to another
tofu state mv \
  module.old_app.aws_instance.web \
  module.new_app.aws_instance.web
```

## Migrating count Index to for_each Key

```bash
# Migrate from count-based to for_each-based addressing
# Before: aws_iam_user.developers[0], [1], [2]
# After:  aws_iam_user.developers["alice"], ["bob"], ["charlie"]

tofu state mv \
  'aws_iam_user.developers[0]' \
  'aws_iam_user.developers["alice"]'

tofu state mv \
  'aws_iam_user.developers[1]' \
  'aws_iam_user.developers["bob"]'

tofu state mv \
  'aws_iam_user.developers[2]' \
  'aws_iam_user.developers["charlie"]'
```

## Moving Entire Modules

```bash
# Move an entire module
tofu state mv module.old_name module.new_name

# Move a module into a nested module
tofu state mv module.database module.application.module.database
```

## Dry Run with -dry-run

```bash
# Preview what would happen without making changes
tofu state mv -dry-run aws_instance.web module.app.aws_instance.web

# Output:
# Would move "aws_instance.web" to "module.app.aws_instance.web"
```

## Moving to a Different State File

```bash
# Move resource from one state file to another
tofu state mv \
  -state=source.tfstate \
  -state-out=destination.tfstate \
  aws_instance.web \
  aws_instance.web

# Split a monolith state into multiple smaller states
tofu state mv \
  -state=monolith.tfstate \
  -state-out=networking.tfstate \
  module.networking.aws_vpc.main \
  aws_vpc.main
```

## Practical Example: Extracting a Module

```bash
# Scenario: extract resources from root into a module

# Step 1: See what's in root
tofu state list | grep -v 'module\.'
# aws_vpc.main
# aws_subnet.public[0]
# aws_subnet.public[1]
# aws_internet_gateway.main

# Step 2: Create the module (modules/networking/main.tf)
# with these resources defined

# Step 3: Move resources to match new addresses
tofu state mv aws_vpc.main module.networking.aws_vpc.main
tofu state mv 'aws_subnet.public[0]' 'module.networking.aws_subnet.public[0]'
tofu state mv 'aws_subnet.public[1]' 'module.networking.aws_subnet.public[1]'
tofu state mv aws_internet_gateway.main module.networking.aws_internet_gateway.main

# Step 4: Verify with plan (should show no changes)
tofu plan
```

## Backup Before Moving

```bash
# state mv creates a backup automatically, but you can also:
cp terraform.tfstate terraform.tfstate.before-mv

# Or use -backup flag
tofu state mv -backup=pre-move.tfstate aws_instance.web aws_instance.app
```

## When to Use moved Blocks Instead

For collaborative teams, prefer `moved` blocks in code:

```hcl
# Prefer this for team environments - changes are tracked in version control
moved {
  from = aws_instance.server
  to   = aws_instance.web_server
}
```

Use `tofu state mv` for:
- One-time emergency fixes
- Working with state directly (no code change wanted)
- Cross-state migrations (different state files)

## Conclusion

`tofu state mv` is a powerful command for refactoring infrastructure without downtime. Always dry-run first, create backups before large operations, and prefer `moved` blocks in code for team environments. Use `state mv` for direct state manipulation, cross-state migrations, and situations where code changes aren't appropriate.
