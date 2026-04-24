# Phase 9 — Testing Infrastructure Code

Most engineers test application code rigorously and treat infrastructure code as untestable. That gap causes production incidents. This phase closes it.

---

## Why Infrastructure Testing Is Different

Testing application code is straightforward — call a function, assert the output. Infrastructure testing has a fundamental constraint:

```
To fully test infrastructure code, you must CREATE real resources.
Real resources cost money and take minutes to provision.
This forces a testing strategy that minimizes expensive tests
while maximizing cheap ones.
```

This is why the testing pyramid matters more for infrastructure than anywhere else.

---

## Topic 9.1 — The Testing Pyramid

```
                        ▲
                       /E\
                      / 2 \          E2E — Terratest
                     /     \         Real infra + behavior validation
                    /───────\        Slowest, most expensive
                   /  Integ  \
                  /           \      Integration — .tftest.hcl apply
                 /             \     Real infra, destroyed after
                /───────────────\    Minutes per test
               /   Plan Tests    \
              /                   \  Plan — .tftest.hcl plan
             /                     \ No real infra, seconds
            /─────────────────────── \
           /         Unit             \
          /                            \ fmt + validate + tfsec + conftest
         /________________________________\ No cloud, runs in seconds
```

**The rule:** spend the most effort at the bottom. Fast, no-cloud tests should catch 80% of issues before a single API call is made.

---

## Topic 9.2 — Unit Layer (No Cloud)

These run without any cloud credentials. They are your first gate.

```bash
# 1. Format check — catches trivial style issues instantly
terraform fmt -check -recursive
# Exit 1 if any file needs formatting
# Fix with: terraform fmt -recursive

# 2. Validate — syntax, type checking, reference validation
terraform validate
# Catches: undefined variables, wrong types, missing required args
# Does NOT catch: invalid AMI IDs, wrong region names — those need API calls

# 3. tfsec — static security analysis
tfsec . --minimum-severity HIGH

# 4. conftest — org policy enforcement
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
conftest test tfplan.json --policy policy/
```

These four together catch a large class of issues in under 30 seconds with zero cloud cost.

---

## Topic 9.3 — Native Terraform Testing (.tftest.hcl)

Added in Terraform 1.6. No external tools, no Go, no extra dependencies. Just `.tftest.hcl` files alongside your module.

### File Structure

```
modules/vpc/
├── main.tf
├── variables.tf
├── outputs.tf
└── tests/
    ├── vpc_basic.tftest.hcl
    └── vpc_multiaz.tftest.hcl
```

### Anatomy of a Test File

```hcl
# modules/vpc/tests/vpc_basic.tftest.hcl

# Override variables for this test run
variables {
  project     = "shopsphere-test"
  environment = "test"
  vpc_cidr    = "10.99.0.0/16"
  az_count    = 2
}

# A run block is one test case
run "vpc_has_correct_cidr" {
  command = plan    # no real infra — fast, free

  assert {
    condition     = aws_vpc.main.cidr_block == "10.99.0.0/16"
    error_message = "VPC CIDR does not match expected value"
  }
}

run "correct_number_of_subnets" {
  command = plan

  assert {
    condition     = length(aws_subnet.public) == 2
    error_message = "Expected 2 public subnets for az_count=2"
  }

  assert {
    condition     = length(aws_subnet.private) == 2
    error_message = "Expected 2 private subnets for az_count=2"
  }
}

run "private_subnets_have_no_public_ip" {
  command = plan

  assert {
    condition = alltrue([
      for subnet in aws_subnet.private :
      subnet.map_public_ip_on_launch == false
    ])
    error_message = "Private subnets must not assign public IPs"
  }
}
```

### `command = plan` vs `command = apply`

```
command = plan
  → evaluates assertions against the PLANNED values
  → no real resources created
  → runs in seconds
  → free
  → limitation: some values are (known after apply) — can't assert on them

command = apply
  → creates REAL resources in AWS
  → assertions run against actual created values
  → Terraform destroys all resources after the test block completes
  → takes minutes, costs money
  → catches things plan can't: wrong IAM permissions, API limits, region issues
```

```hcl
# Integration test — creates real infra, validates real outputs
run "vpc_creates_successfully" {
  command = apply    # real resources

  assert {
    condition     = aws_vpc.main.id != ""
    error_message = "VPC was not created — ID is empty"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames must be enabled"
  }
}
# After this run block: Terraform automatically destroys aws_vpc.main
# and everything created by this test
```

---

### Multiple Run Blocks — Stateful Tests

Run blocks within a test file share state — resources created in earlier blocks exist for later blocks:

```hcl
# tests/compute.tftest.hcl

variables {
  project     = "shopsphere-test"
  environment = "test"
  vpc_cidr    = "10.99.0.0/16"
}

# Block 1 — create the VPC first (dependency)
run "create_vpc" {
  command = apply
  module {
    source = "../../modules/vpc"
  }
}

# Block 2 — use VPC outputs to test compute module
run "create_compute" {
  command = apply

  variables {
    vpc_id             = run.create_vpc.vpc_id            # reference earlier run
    private_subnet_ids = run.create_vpc.private_subnet_ids
  }

  assert {
    condition     = aws_instance.app[0].subnet_id == run.create_vpc.private_subnet_ids[0]
    error_message = "App instance must be in private subnet"
  }
}
# After all blocks: everything is destroyed
```

---

### Running Tests

```bash
# Run all tests in current directory and subdirectories
terraform test

# Run specific test file
terraform test -filter=tests/vpc_basic.tftest.hcl

# Run with verbose output
terraform test -verbose

# Output:
# vpc_basic.tftest.hcl... in progress
#   run "vpc_has_correct_cidr"... pass
#   run "correct_number_of_subnets"... pass
#   run "private_subnets_have_no_public_ip"... pass
# vpc_basic.tftest.hcl... tearing down
# vpc_basic.tftest.hcl... pass
#
# Success! 3 passed, 0 failed.
```

---

## Topic 9.4 — Mock Providers (Terraform 1.7+)

Testing with `command = apply` costs money and takes time. Mock providers let you test apply-time behavior without real cloud calls:

```hcl
# tests/vpc_mock.tftest.hcl

# Override the AWS provider with a mock
mock_provider "aws" {
  mock_resource "aws_vpc" {
    defaults = {
      id         = "vpc-mock12345"
      arn        = "arn:aws:ec2:ap-south-1:123456789:vpc/vpc-mock12345"
      cidr_block = "10.99.0.0/16"
    }
  }

  mock_resource "aws_subnet" {
    defaults = {
      id                      = "subnet-mock12345"
      vpc_id                  = "vpc-mock12345"
      map_public_ip_on_launch = false
    }
  }
}

run "vpc_outputs_are_correct" {
  command = apply    # applies against MOCK provider — no real AWS calls

  assert {
    condition     = output.vpc_id == "vpc-mock12345"
    error_message = "VPC ID output is wrong"
  }
}
```

**When to use mock providers:**

```
Real apply tests    → validating actual AWS behavior, IAM permissions
Mock apply tests    → validating module logic, output structure,
                      conditional resource creation
                      anything where you're testing YOUR code,
                      not AWS's behavior
```

---

## Topic 9.5 — Terratest (Go-Based E2E Testing)

Terratest sits at the top of the pyramid — real infra, real behavior validation. It goes beyond what native Terraform testing can check:

```
Native .tftest.hcl              Terratest
────────────────────────────────────────────────────────────
Terraform attribute assertions   HTTP request to running service
Resource attribute checks        SSH into instance, run command
Output value validation          Verify app actually responds
Module logic testing             Load test the infrastructure
                                 Multi-step deployment validation
                                 Cross-service integration checks
```

### Basic Terratest Example

```go
// test/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestVPCModule(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        // Path to module being tested
        TerraformDir: "../modules/vpc",

        Vars: map[string]interface{}{
            "project":     "shopsphere-test",
            "environment": "test",
            "vpc_cidr":    "10.99.0.0/16",
            "az_count":    2,
        },
    }

    // defer runs cleanup even if test fails — always destroys
    defer terraform.Destroy(t, terraformOptions)

    // Apply the module
    terraform.InitAndApply(t, terraformOptions)

    // Get outputs
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")

    // Validate using AWS SDK directly
    vpc := aws.GetVpcById(t, vpcID, "ap-south-1")
    assert.Equal(t, "10.99.0.0/16", aws.GetCidrBlockOfVpc(t, vpc))

    // Validate subnets exist in correct AZs
    publicSubnetIDs := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
    assert.Equal(t, 2, len(publicSubnetIDs))
}

func TestALBResponds(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../envs/test",
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    albDNS := terraform.Output(t, terraformOptions, "alb_dns_name")

    // Actually hit the load balancer — this is what .tftest.hcl cannot do
    url := fmt.Sprintf("http://%s/actuator/health", albDNS)
    http_helper.HttpGetWithRetry(
        t,
        url,
        nil,
        200,                  // expected status code
        "",                   // expected body (empty = any)
        10,                   // max retries
        10*time.Second,       // sleep between retries
    )
}
```

```bash
# Run Terratest
cd test
go test -v -timeout 30m ./...

# Run specific test
go test -v -run TestVPCModule -timeout 30m ./...
```

---

## Topic 9.6 — Testing the `prevent_destroy` Problem

A module with `prevent_destroy = true` cannot be destroyed in tests — Terraform test cleanup will fail. The solution:

```hcl
# In your module — parameterize the lifecycle
variable "enable_prevent_destroy" {
  type    = bool
  default = true
  description = "Set to false in test environments"
}

# Unfortunately lifecycle blocks don't support variable expressions directly
# The pattern is to use separate resources or separate modules for test vs prod

# Alternative: use a feature flag via a separate resource
resource "aws_db_instance" "main" {
  # ... config
}

# Separate resource just for protection — skip in tests
resource "null_resource" "db_protect" {
  count = var.enable_prevent_destroy ? 1 : 0

  lifecycle {
    prevent_destroy = true
  }

  # This resource being present prevents accidental destroy of the stack
  # In tests: count = 0, so it doesn't exist, cleanup works fine
}
```

The cleanest solution: in test configs, pass `enable_prevent_destroy = false`. In prod, the default `true` applies.

---

## Phase 9 — Testing Strategy for ShopSphere

Mapping the pyramid to your actual modules:

```
modules/vpc/tests/
├── vpc_unit.tftest.hcl          # plan tests — cidr, subnet count, tags
├── vpc_integration.tftest.hcl   # apply tests — real VPC, verify IDs non-empty
└── vpc_mock.tftest.hcl          # mock apply — output structure validation

modules/compute/tests/
├── compute_unit.tftest.hcl      # plan tests — SG rules, instance config
└── compute_integration.tftest.hcl

test/                            # Terratest — top of pyramid
├── vpc_test.go                  # VPC + subnet validation via AWS SDK
├── alb_test.go                  # ALB responds to HTTP requests
└── e2e_test.go                  # full stack: request in → response out
```

**CI integration:**

```yaml
# In your pipeline — test stage before plan
- name: Run unit tests
  run: terraform test -filter=tests/*_unit.tftest.hcl
  # Fast, no cloud, runs on every PR

- name: Run integration tests
  run: terraform test -filter=tests/*_integration.tftest.hcl
  # Slower, creates real infra, runs nightly or on merge to main
  # Never on every PR — too slow and expensive
```

---

## Breaking Exercises

```bash
# Exercise 1 — Write tests for your VPC module
# Minimum: 3 plan tests covering
#   - correct CIDR
#   - correct subnet count based on az_count variable
#   - private subnets do not assign public IPs
# Run: terraform test
# Break one assertion intentionally → observe failure message

# Exercise 2 — Write a mock provider test
# Test that your module's outputs are correct
# without making any real AWS calls
# Validate: vpc_id output is non-empty string

# Exercise 3 — Test the destroy lifecycle
# Write an apply test for a module with prevent_destroy
# Observe the cleanup failure
# Fix it using the parameterized lifecycle pattern
# Run terraform test again — observe clean cleanup
```

---

## Phase 9 Interview Questions

1. What is the difference between `command = plan` and `command = apply` in a `.tftest.hcl` run block?
2. What does Terratest test that native Terraform testing cannot?
3. Why must infrastructure tests always clean up (destroy) after themselves?
4. What is the testing pyramid for infrastructure code?
5. How do you test a module that has `prevent_destroy` on one of its resources?

---

Say **"move"** for Phase 10 — Enterprise Patterns, or **"drill"** for interview questions.
