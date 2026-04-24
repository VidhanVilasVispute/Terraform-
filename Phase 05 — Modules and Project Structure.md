
# Phase 5 — Modules and Project Structure

Stop dumping everything in one directory. This phase is about structuring Terraform like a production engineering team.

---

## Topic 5.1 — What a Module Actually Is

A module is just a **directory of `.tf` files**. That's it. Every Terraform config you've written so far is already a module — the root module.

The difference between a root module and a child module:

```
Root module   → what you run terraform apply from
               has its own state, its own backend
               calls child modules

Child module  → reusable unit of infrastructure
               takes inputs (variables)
               exposes outputs
               has NO state of its own — state lives in the root module
```

The contract of a well-designed module:

```
┌─────────────────────────────────────────────────┐
│                   MODULE                        │
│                                                 │
│  inputs (variables)  ──►  resources  ──►  outputs│
│                                                 │
│  - no hardcoded values                          │
│  - no hardcoded region                          │
│  - reusable across environments                 │
└─────────────────────────────────────────────────┘
```

---

### Module File Structure

```
modules/
└── vpc/
    ├── main.tf        # all resources
    ├── variables.tf   # inputs — everything configurable goes here
    ├── outputs.tf     # what callers can consume
    └── README.md      # what it does, inputs, outputs, examples
```

**variables.tf — inputs into the module:**

```hcl
variable "project" {
  type        = string
  description = "Project name used for resource naming"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "az_count" {
  type        = number
  description = "Number of availability zones to deploy across"
  default     = 2
}
```

**outputs.tf — what callers consume:**

```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID"
}

output "public_subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "List of public subnet IDs"
}

output "private_subnet_ids" {
  value       = aws_subnet.private[*].id
  description = "List of private subnet IDs"
}

output "nat_gateway_ids" {
  value       = aws_nat_gateway.main[*].id
  description = "List of NAT Gateway IDs"
}
```

---

### Calling a Module

```hcl
# envs/dev/main.tf

module "vpc" {
  source = "../../modules/vpc"   # relative path to module directory

  project     = var.project
  environment = var.environment
  vpc_cidr    = "10.0.0.0/16"
  az_count    = 2
}

module "compute" {
  source = "../../modules/compute"

  project            = var.project
  environment        = var.environment
  vpc_id             = module.vpc.vpc_id               # consume vpc output
  private_subnet_ids = module.vpc.private_subnet_ids   # consume vpc output
  public_subnet_ids  = module.vpc.public_subnet_ids
}
```

**Module reference syntax:**

```hcl
module.vpc.vpc_id              # output from module "vpc"
module.compute.alb_dns_name    # output from module "compute"
module.vpc.private_subnet_ids[0]  # index into list output
```

After adding a new module or changing module source, always run:

```bash
terraform init   # downloads/updates module source
```

---

## Topic 5.2 — Environment-Based Project Structure

This is the production standard. Each environment is a completely independent Terraform root module with its own state.

```
shopsphere-infra/
├── modules/                    # reusable building blocks
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── rds/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── eks/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── envs/
│   ├── dev/
│   │   ├── main.tf           # calls modules with dev values
│   │   ├── backend.tf        # points to dev state file
│   │   ├── variables.tf
│   │   └── terraform.tfvars  # dev-specific values
│   ├── staging/
│   │   ├── main.tf           # same modules, staging values
│   │   ├── backend.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf           # same modules, prod values
│       ├── backend.tf
│       ├── variables.tf
│       └── terraform.tfvars
│
└── global/
    └── s3-backend/           # bootstraps the S3 + DynamoDB backend
        ├── main.tf
        └── terraform.tfvars
```

### What Each Environment Contains

**envs/dev/backend.tf:**
```hcl
terraform {
  backend "s3" {
    bucket         = "shopsphere-terraform-state"
    key            = "dev/terraform.tfstate"      # dev-specific key
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

**envs/prod/backend.tf:**
```hcl
terraform {
  backend "s3" {
    bucket         = "shopsphere-terraform-state"
    key            = "prod/terraform.tfstate"     # completely isolated
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

**envs/dev/terraform.tfvars:**
```hcl
project       = "shopsphere"
environment   = "dev"
vpc_cidr      = "10.0.0.0/16"
instance_type = "t3.micro"
az_count      = 2
```

**envs/prod/terraform.tfvars:**
```hcl
project       = "shopsphere"
environment   = "prod"
vpc_cidr      = "10.1.0.0/16"    # different CIDR — no overlap
instance_type = "t3.medium"      # bigger instances
az_count      = 3                # more AZs for resilience
```

**The key property:** destroying dev state has zero effect on prod. They are completely isolated — different state files, different resources, different AWS accounts in mature setups.

---

## Topic 5.3 — Loops and Dynamic Blocks in Modules

### `for_each` on a Map of Objects

The most flexible pattern for creating multiple related resources:

```hcl
# variables.tf — caller passes in a map of service configs
variable "services" {
  type = map(object({
    port          = number
    instance_type = string
    min_size      = number
    max_size      = number
  }))
}
```

```hcl
# terraform.tfvars — ShopSphere services as a map
services = {
  "order-service" = {
    port          = 8081
    instance_type = "t3.small"
    min_size      = 2
    max_size      = 5
  }
  "product-service" = {
    port          = 8082
    instance_type = "t3.micro"
    min_size      = 1
    max_size      = 3
  }
  "user-service" = {
    port          = 8083
    instance_type = "t3.micro"
    min_size      = 1
    max_size      = 3
  }
}
```

```hcl
# main.tf — one security group per service, driven entirely by the map
resource "aws_security_group" "services" {
  for_each = var.services

  name        = "${local.name_prefix}-${each.key}-sg"
  description = "SG for ${each.key}"
  vpc_id      = var.vpc_id

  tags = {
    Name    = "${local.name_prefix}-${each.key}-sg"
    Service = each.key
  }
}

resource "aws_security_group_rule" "service_ingress" {
  for_each = var.services

  type                     = "ingress"
  from_port                = each.value.port
  to_port                  = each.value.port
  protocol                 = "tcp"
  source_security_group_id = var.alb_security_group_id
  security_group_id        = aws_security_group.services[each.key].id
}
```

Adding a new service = add one entry to the map. Zero code changes. Zero risk to existing services.

---

### Dynamic Blocks

For when a resource has a **repeating nested block** that varies per instance:

**Without dynamic blocks — repetitive and brittle:**
```hcl
resource "aws_security_group" "app" {
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}
```

**With dynamic blocks — driven by a variable:**
```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { port = 80,   protocol = "tcp", cidr_blocks = ["0.0.0.0/0"]    },
    { port = 443,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"]    },
    { port = 8080, protocol = "tcp", cidr_blocks = ["10.0.0.0/8"]   },
  ]
}

resource "aws_security_group" "app" {
  dynamic "ingress" {
    for_each = var.ingress_rules      # iterator variable name = block label
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

`ingress.value` refers to the current element. `ingress.key` refers to the current index (for lists) or key (for maps).

---

## Topic 5.4 — Workspaces vs Folder-Based

This comes up in interviews. Know the clear position.

### Workspaces

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
terraform apply
```

Each workspace gets its own state file in the backend. Looks convenient.

**Where workspaces fall apart:**

```
Problem 1 — No provider config isolation
  You can't have different AWS accounts per workspace
  Dev and prod would use the same account — dangerous

Problem 2 — No infrastructure shape differences
  What if prod needs 3 AZs and dev needs 2?
  What if prod has RDS Multi-AZ and dev has single instance?
  Workspaces force identical config, just different values

Problem 3 — Human error surface area
  terraform workspace select prod  ← easy to forget
  terraform apply                  ← you just applied to wrong env
  Folder structure makes the env explicit in the path you cd into

Problem 4 — CI/CD complexity
  Parameterizing workspace name across pipelines is messier
  than just having separate pipeline steps per env folder
```

**The verdict:**

```
Workspaces      → short-lived feature branches, simple identical envs
Folder-based    → production standard, real environment isolation
```

If an interviewer asks: "We use workspaces, what do you think?" — acknowledge the valid use cases (feature environments, simple cases), then explain the limitations for real production isolation.

---

## ShopSphere Module Design

Here's what the ShopSphere infra module structure should look like as it grows:

```
shopsphere-infra/
├── modules/
│   ├── vpc/              # network layer — VPC, subnets, IGW, NAT, route tables
│   ├── security/         # security groups — one per service tier
│   ├── compute/          # EC2, ALB, target groups, ASG
│   ├── rds/              # PostgreSQL — per service database
│   ├── elasticache/      # Redis — session store, cache
│   ├── msk/              # Kafka — event streaming
│   ├── elasticsearch/    # search service
│   ├── eks/              # Kubernetes cluster + node groups
│   └── iam/              # roles, policies, instance profiles
│
├── envs/
│   ├── dev/
│   └── prod/
│
└── global/
    └── s3-backend/
```

Each module owns one concern. `rds` module doesn't know about `compute`. `eks` module consumes `vpc` outputs. Clean, testable, independently evolvable.

---

## Phase 5 — Hands-On Deliverable

Refactor the Phase 4 config into this structure:

```
shopsphere-infra/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── compute/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── envs/
    └── dev/
        ├── main.tf        # calls modules
        ├── backend.tf
        └── terraform.tfvars
```

**The test:** after refactoring, run:

```bash
terraform state mv aws_instance.web module.compute.aws_instance.web
terraform state mv aws_security_group.web module.compute.aws_security_group.web
# ... move all resources to their new module addresses

terraform plan
# Must show: No changes. Infrastructure is up-to-date.
```

Zero destroys. Zero recreates. Same infrastructure, better structure.

---

## Phase 5 Interview Questions

1. What is the difference between a root module and a child module?
2. Why is folder-based environment isolation preferred over workspaces for production?
3. How do you reference a resource in module A from module B in the same root module?
4. What is `for_each` on a map of objects? Give a concrete example where you'd use it.
5. What problem do dynamic blocks solve? Why not just write the blocks manually?

---

Say **"move"** for Phase 6 — Provider and Resource Internals, or **"drill"** for the interview questions.
