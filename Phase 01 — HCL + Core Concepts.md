# Phase 1 — HCL + Core Concepts

---

## Topic 1.1 — The Six Block Types

Terraform has exactly **six block types**. Everything you ever write is one of these.

```
┌─────────────┬──────────────────────────────────────────────────┐
│   Block     │  Purpose                                         │
├─────────────┼──────────────────────────────────────────────────┤
│  resource   │  Creates/manages real infrastructure             │
│  provider   │  Configures the cloud (AWS, GCP...)              │
│  variable   │  Input into your config                          │
│  output     │  Values you export out                           │
│  locals     │  Computed values internal to your module         │
│  data       │  Read-only lookup of EXISTING infrastructure     │
└─────────────┴──────────────────────────────────────────────────┘
```

Let's go through each one properly.

---

### `resource` — The Core Block

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
}
```

Structure is always: `resource "<TYPE>" "<NAME>"`. The combination `aws_instance.web` is how you reference this resource anywhere else in your config.

`TYPE` comes from the provider — `aws_instance`, `aws_s3_bucket`, `aws_security_group`.  
`NAME` is yours — it only exists inside Terraform, not in AWS.

---

### `provider` — Cloud Configuration

```hcl
provider "aws" {
  region = "ap-south-1"
}
```

Tells Terraform which cloud and how to authenticate. You can have multiple provider blocks with aliases — we'll cover that in Phase 6.

---

### `variable` — Inputs

```hcl
variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "EC2 instance type"

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Only approved instance types allowed."
  }
}
```

Variables make your config reusable. Without them, you're hardcoding — and hardcoded configs break the moment you need dev vs prod.

`validation` blocks run **before** any API calls. Fail fast, fail cheap.

---

### `output` — Exports

```hcl
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of the web server"
}
```

Two real uses:
- **Human readable** — printed after `terraform apply`, useful for knowing what got created
- **Module chaining** — one module exposes outputs, another module consumes them

```hcl
# module A exposes this
output "vpc_id" {
  value = aws_vpc.main.id
}

# module B consumes it
subnet_id = module.vpc.vpc_id
```

---

### `locals` — Internal Computed Values

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

**locals vs variables — the key difference:**

| | `variable` | `locals` |
|---|---|---|
| Set by | caller (from outside) | computed inside the module |
| Purpose | input from user/environment | reusable internal expressions |
| Override | yes, from CLI or tfvars | no, internal only |

Use `locals` when you're computing something from other values and don't want to repeat that expression 10 times.

---

### `data` — Read Existing Infrastructure

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical's AWS account

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

# Reference it
resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
}
```

**Critical distinction — `data` vs `resource`:**

```
resource  →  Terraform OWNS this. It creates, updates, destroys it.
data      →  Terraform just READS this. It never modifies it.
```

`data` is for things that already exist and you just need to reference — latest AMI, an existing VPC you didn't create with Terraform, a secret from Secrets Manager.

**When does each execute?**
- `resource` blocks execute during `apply`
- `data` blocks execute during `plan` (they need to resolve before the diff is calculated)

---

## Topic 1.2 — Arguments vs Attributes

This trips people up constantly in interviews.

```
Arguments  →  you SET them (inputs going IN to a resource)
Attributes →  Terraform reads them BACK after creation (outputs coming OUT)
```

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef"   # ← ARGUMENT: you set this
  instance_type = "t3.micro"      # ← ARGUMENT: you set this
}

# After creation, Terraform reads back:
aws_instance.web.id          # ← ATTRIBUTE: assigned by AWS
aws_instance.web.public_ip   # ← ATTRIBUTE: assigned by AWS
aws_instance.web.arn         # ← ATTRIBUTE: assigned by AWS
```

You **cannot** do `aws_instance.web.ami` and expect it to work as a reference to give to another resource — you reference **attributes**, not arguments.

When you see something like `source_security_group_id = aws_security_group.app.id` — `.id` is an attribute. It didn't exist until AWS created the security group and assigned it an ID.

---

## Topic 1.3 — Expressions and Functions

### String Interpolation
```hcl
name = "${var.project}-${var.environment}-web"
# If project=shopsphere, environment=prod → "shopsphere-prod-web"
```

### Conditional Expression
```hcl
instance_type = var.environment == "prod" ? "t3.medium" : "t3.micro"
```

Same ternary pattern as Java. Works anywhere an expression is valid.

### Functions You Must Know Cold

```hcl
# length — count elements
length(var.subnets)          # [a, b, c] → 3

# toset — deduplicate a list, convert to set
toset(["a", "b", "a"])       # → {"a", "b"}

# flatten — collapse nested lists
flatten([[1,2],[3,4]])        # → [1, 2, 3, 4]

# merge — combine maps (second map wins on conflict)
merge(
  { env = "prod", team = "platform" },
  { env = "staging" }
)
# → { env = "staging", team = "platform" }

# lookup — safe map access with default
lookup(var.instance_types, "prod", "t3.micro")
# if "prod" key doesn't exist, returns "t3.micro"

# cidrsubnet — subnet math
cidrsubnet("10.0.0.0/16", 8, 1)   # → "10.0.1.0/24"
cidrsubnet("10.0.0.0/16", 8, 2)   # → "10.0.2.0/24"
# You'll use this heavily when building VPCs
```

---

## Topic 1.4 — `count` vs `for_each`

This is one of the most important decisions you make. Get it wrong and you cause unintended destroy/recreate cycles in production.

### `count` — Index Based

```hcl
resource "aws_instance" "web" {
  count         = 3
  instance_type = "t3.micro"
  ami           = var.ami_id
}

# Creates:
# aws_instance.web[0]
# aws_instance.web[1]
# aws_instance.web[2]
```

**The problem with count:**

```
Initial:  ["prod", "staging", "dev"]
           [0]       [1]        [2]

Remove "staging":  ["prod", "dev"]
                    [0]      [1]

Terraform sees:
  web[1] changed  → staging→dev  (UPDATE or REPLACE)
  web[2] deleted  → destroy dev  ← WRONG, you wanted to delete staging
```

Removing any element that isn't the last one shifts all indices. Terraform interprets this as changes to every resource after the removed one. In production, this causes unwanted destroys.

### `for_each` — Key Based

```hcl
resource "aws_instance" "web" {
  for_each      = toset(["prod", "staging", "dev"])
  instance_type = "t3.micro"
  ami           = var.ami_id

  tags = {
    Name = "shopsphere-${each.key}"
  }
}

# Creates:
# aws_instance.web["prod"]
# aws_instance.web["staging"]
# aws_instance.web["dev"]
```

**Remove "staging" now:**
```
aws_instance.web["prod"]    → untouched
aws_instance.web["staging"] → destroyed (exactly this one)
aws_instance.web["dev"]     → untouched
```

Keys are stable. Only the removed key is affected.

**Rule:** Use `for_each` for anything with names. Use `count` only for truly identical anonymous resources where order never changes.

**One gotcha:** `for_each` requires a `map` or `set` — not a plain `list`. If you have a `list(string)`, convert it:

```hcl
for_each = toset(var.environment_list)
```

---

## Topic 1.5 — Variable Precedence

From **highest** to **lowest**:

```
1. CLI flags          → terraform apply -var="env=prod"
2. *.auto.tfvars      → prod.auto.tfvars (auto-loaded)
3. terraform.tfvars   → auto-loaded if present
4. TF_VAR_ env vars   → export TF_VAR_env=prod
5. default in block   → default = "dev"
```

Higher priority overrides lower. This is how you have one config serve dev and prod — `terraform.tfvars` sets defaults, CI pipeline overrides with `-var` flags.

---

## Phase 1 — Hands-On Deliverable

Here's your first real config to build. This is a ShopSphere-anchored EC2 + Security Group:

**File structure:**
```
shopsphere-infra/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

**variables.tf**
```hcl
variable "project" {
  type        = string
  description = "Project name"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Only approved instance types allowed."
  }
}

variable "ami_id" {
  type        = string
  description = "AMI ID for EC2 instance"
}
```

**main.tf**
```hcl
provider "aws" {
  region = "ap-south-1"
}

locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_security_group" "web" {
  name        = "${local.name_prefix}-web-sg"
  description = "Security group for ShopSphere web tier"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = local.common_tags
}

resource "aws_instance" "web" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
  })
}
```

**outputs.tf**
```hcl
output "instance_id" {
  value       = aws_instance.web.id
  description = "EC2 instance ID"
}

output "instance_public_ip" {
  value       = aws_instance.web.public_ip
  description = "Public IP of web instance"
}

output "security_group_id" {
  value       = aws_security_group.web.id
  description = "Security group ID"
}
```

**terraform.tfvars**
```hcl
project     = "shopsphere"
environment = "dev"
ami_id      = "ami-0f58b397bc5c1f2e8"  # Ubuntu 22.04 ap-south-1
```

---

### Breaking Exercise (Mandatory)

After the basic config works, do these:

**Break 1:** Set `environment = "uat"` in tfvars → run `terraform plan` → observe validation error

**Break 2:** Change `for_each = toset(var.some_list)` where `some_list` is a `list(string)` without `toset()` → observe the error → fix it

**Break 3:** Try to reference `aws_instance.web.ami` as a value in an output → run plan → observe the behavior and understand why it works or doesn't

---

## Phase 1 Interview Questions

Before we move to Phase 2, make sure you can answer these cold:

1. What is the difference between a `data` source and a `resource`? When does each execute?
2. When would you use `locals` instead of `variables`?
3. What is the concrete danger of using `count` for a list of named resources?
4. A `for_each` over a `list(string)` fails — why, and how do you fix it?
5. What does `sensitive = true` on an output do? Does it prevent the value from appearing in state?

---

Take your time with the hands-on. When you're done — or if you want to drill the interview questions first — say **"Phase 1 done"** or **"drill questions"** and we move to Phase 2.
