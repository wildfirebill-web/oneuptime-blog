# How to Use Terraform Data Sources to Read Portainer Resources (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Terraform, Data Source, Infrastructure as Code, DevOps

Description: Learn how to use Terraform data sources to read existing Portainer resources and reference them without managing their lifecycle.

## What Are Terraform Data Sources?

Data sources let you read existing resources without managing (creating/deleting) them. This is useful when:

- You want to reference an existing environment without importing it.
- Multiple Terraform workspaces need to share resource references.
- You want to look up dynamic IDs without hardcoding them.

## Reading Environments

```hcl
# Read an existing Portainer environment by name

data "portainer_environment" "production" {
  name = "production"
}

# Use the environment ID in other resources
resource "portainer_stack" "myapp" {
  name        = "my-app"
  # Reference the data source instead of hardcoding the ID
  endpoint_id = data.portainer_environment.production.id
}

output "production_endpoint_id" {
  value = data.portainer_environment.production.id
}
```

## Reading Users

```hcl
# Look up an existing user by username
data "portainer_user" "admin" {
  username = "admin"
}

data "portainer_user" "alice" {
  username = "alice.smith"
}

output "alice_user_id" {
  value = data.portainer_user.alice.id
}
```

## Reading Teams

```hcl
# Look up an existing team by name
data "portainer_team" "devops" {
  name = "devops"
}

# Add a new user to the existing team
resource "portainer_team_membership" "new_member" {
  team_id = data.portainer_team.devops.id
  user_id = portainer_user.new_hire.id
  role    = 2
}
```

## Reading Registries

```hcl
# Read an existing registry to use its ID
data "portainer_registry" "harbor" {
  name = "Company Harbor"
}

# Associate the registry with a new environment
resource "portainer_environment_registry" "prod_harbor" {
  environment_id = portainer_environment.new_cluster.id
  registry_id    = data.portainer_registry.harbor.id
}
```

## Cross-Workspace References

```hcl
# In the "shared" workspace, infrastructure is defined
# In the "app-team" workspace, reference shared infrastructure

data "terraform_remote_state" "shared" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "portainer/shared/terraform.tfstate"
    region = "us-east-1"
  }
}

# Reference the environment ID from the shared workspace
resource "portainer_stack" "app_team_stack" {
  name        = "app-team-app"
  # Use the output from the shared workspace
  endpoint_id = data.terraform_remote_state.shared.outputs.production_environment_id
}
```

## Combining Data Sources with Resources

```hcl
# Read existing infrastructure, create new resources on top

# Read existing production environment
data "portainer_environment" "production" {
  name = "production"
}

# Read existing devops team
data "portainer_team" "devops" {
  name = "devops"
}

# Create a new stack in the existing environment
resource "portainer_stack" "new_service" {
  name               = "new-service"
  endpoint_id        = data.portainer_environment.production.id
  stack_file_content = file("stacks/new-service/docker-compose.yml")
}

# Grant the existing devops team access to the new environment we create
resource "portainer_environment" "new_env" {
  name = "new-environment"
  type = 1
  url  = "tcp://new-host:9001"
}
```

## Listing All Available Environments

```hcl
# List all environments (returns a list of environment objects)
data "portainer_environments" "all" {}

output "all_environment_names" {
  value = [for env in data.portainer_environments.all.environments : env.name]
}
```

## Conclusion

Terraform data sources for Portainer enable clean separation between resources that are managed in different workspaces or by different teams. Use them to reference existing environments and users without creating circular dependencies between Terraform configurations.
