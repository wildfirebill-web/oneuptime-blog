# OpenTofu Checks vs. Postconditions: What's the Difference

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Checks, Postconditions, Validation, Infrastructure as Code

Description: Learn the difference between OpenTofu check blocks and postconditions, and when to use each for infrastructure validation.

---

OpenTofu provides two mechanisms for validating infrastructure state: `postcondition` blocks inside resource `lifecycle` and standalone `check` blocks. They serve different purposes and run at different points in the workflow.

---

## Postconditions

`postcondition` blocks validate resource attributes after a resource is created or updated. If the condition fails, the entire apply is aborted.

```hcl
resource "aws_lb" "main" {
  name               = "main-lb"
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id

  lifecycle {
    postcondition {
      condition     = self.dns_name != ""
      error_message = "Load balancer did not receive a DNS name after creation."
    }
  }
}
```

Key characteristics:
- Runs during `apply`, after the resource is created/updated
- Failure aborts the apply with an error
- Has access to `self` (the resource's own attributes)
- Suitable for validating that cloud-assigned values meet requirements

---

## Check Blocks

`check` blocks validate arbitrary data sources or conditions and report warnings without aborting. They're used for ongoing health assertions.

```hcl
check "lb_health" {
  data "http" "lb_health_check" {
    url = "http://${aws_lb.main.dns_name}/health"
  }

  assert {
    condition     = data.http.lb_health_check.status_code == 200
    error_message = "Load balancer health check returned non-200 status."
  }
}
```

Key characteristics:
- Runs at the end of `plan` and `apply`
- Failures produce **warnings**, not errors - apply still succeeds
- Can use `data` sources to fetch external state
- Suitable for monitoring, compliance checks, and soft assertions

---

## Side-by-Side Comparison

| Feature              | `postcondition`           | `check` block              |
|----------------------|---------------------------|----------------------------|
| Placement            | Inside `lifecycle`        | Top-level block             |
| Runs during          | Apply (after create/update)| Plan and apply             |
| On failure           | Aborts apply              | Warns, continues apply      |
| Access to `self`     | Yes                       | No                          |
| Can use data sources | No                        | Yes                         |
| Use case             | Hard resource requirements | Soft health/compliance checks|

---

## When to Use Each

Use `postcondition` when:
- A cloud-assigned attribute must meet a requirement (e.g., endpoint not empty)
- A failure should block the deployment

Use `check` blocks when:
- You want visibility into external health without blocking deploys
- Implementing compliance assertions that should warn but not break CI/CD
- Running `tofu plan` in continuous validation mode

---

## Summary

`postcondition` is a hard gate inside a resource's `lifecycle` that aborts apply on failure. `check` blocks are soft assertions that run at plan/apply time and warn without aborting. Use postconditions to enforce hard requirements on resource attributes, and check blocks for ongoing health and compliance monitoring.
