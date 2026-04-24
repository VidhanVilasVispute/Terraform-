# Phase 6 — Provider and Resource Internals

Most engineers know how to use providers. Very few know what's actually happening underneath. This phase gives you that depth.

---

## Topic 6.1 — What a Provider Actually Is

A provider is a **separate binary** — not part of Terraform core. When you run `terraform init`, Terraform downloads it from the registry and stores it locally:

```
.terraform/
└── providers/
    └── registry.terraform.io/
        └── hashicorp/
            └── aws/
                └── 5.40.0/
                    └── linux_amd64/
                        └── terraform-provider-aws_v5.40.0  ← actual binary
```

This binary contains three things:

```
┌──────────────────────────────────────────────────────────┐
│                    PROVIDER BINARY                       │
│                                                          │
│  1. SCHEMA                                               │
│     Every resource type, every attribute,                │
│     which are required/optional, which types,            │
│     which attributes FORCE REPLACEMENT on change         │
│                                                          │
│  2. CRUD LOGIC                                           │
│     Create → POST /ec2/run-instances                     │
│     Read   → GET  /ec2/describe-instances                │
│     Update → PUT  /ec2/modify-instance-attribute         │
│     Delete → POST /ec2/terminate-instances               │
│                                                          │
│  3. AUTH HANDLING                                        │
│     AWS credential chain, STS assume role,               │
│     token refresh, retry logic                           │
└──────────────────────────────────────────────────────────┘
```

**Why this matters for debugging:**

When a `terraform plan` shows `-/+ forces replacement` on an attribute you barely changed — that decision lives in the **provider schema**, not in your config and not in Terraform core. To understand why, you look at the provider source code or changelog — not Terraform documentation.

---

### Version Constraints

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"      # 5.x only — allows 5.1, 5.40, not 6.0
    }
  }
}
```

Constraint operators:

```
~> 5.0          → >= 5.0, < 6.0   (patch + minor updates, lock major)
~> 5.40         → >= 5.40, < 5.41  (patch updates only, lock minor)
>= 4.0, < 6.0  → explicit range
= 5.40.0        → exact pin — use when a specific version fixed a bug
```

**The `.terraform.lock.hcl` file:**

```hcl
# .terraform.lock.hcl — auto-generated, commit this to git
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.40.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",   # cryptographic hash of the binary
  ]
}
```

This locks the **exact** provider version across all team members and CI pipelines. Without it, two engineers could run `terraform init` and get different provider versions — subtle bugs and plan diffs for no reason.

```bash
terraform init -upgrade    # explicitly upgrade to latest within constraints
```

---

### Provider Internals Debugging

When you hit unexpected behavior, check the provider changelog before anything else:

```bash
# See exactly what API calls the provider is making
TF_LOG=DEBUG terraform plan 2>&1 | grep -i "aws"

# Typical debug output shows:
# [DEBUG] provider.terraform-provider-aws: 2024/01/15 calling EC2 DescribeInstances
# [DEBUG] provider.terraform-provider-aws: 2024/01/15 response: {Reservations: [...]}
```

Real skill: knowing whether a bug is **your config**, **provider behavior**, or **AWS API behavior**. The provider changelog tells you what changed in each version — this is how you diagnose "it worked last week but broke today after someone ran `terraform init -upgrade`."

---

## Topic 6.2 — Importing Existing Infrastructure

The scenario: your company has been running infrastructure manually for years. You need to bring it under Terraform management without destroying and recreating anything.

### The Full Import Workflow

```
Step 1 — Find the resource ID in AWS
         EC2:   i-0abc1234567890def
         SG:    sg-0abc123
         VPC:   vpc-0abc123
         RDS:   my-database-identifier
         S3:    bucket-name (just the name)

Step 2 — Write the resource block in config
         (can be empty initially — Terraform will populate state)

Step 3 — Run terraform import

Step 4 — Run terraform plan and observe the diff

Step 5 — Update config until plan shows zero diff
```

```hcl
# Step 2 — write the skeleton resource block
resource "aws_instance" "web" {
  # attributes will come from import — fill these after
}
```

```bash
# Step 3 — import using real AWS resource ID
terraform import aws_instance.web i-0abc1234567890def

# Output:
# aws_instance.web: Importing from ID "i-0abc1234567890def"
# aws_instance.web: Import prepared!
# aws_instance.web: Refreshing state... [id=i-0abc1234567890def]
# Import successful!
```

```bash
# Step 4 — see what Terraform wants to change
terraform plan

# Will show diffs between your empty/partial config
# and what actually exists in AWS

# Step 5 — fill in your config to match reality
# Run plan again → repeat until:
# No changes. Infrastructure is up-to-date.
```

**The most common import mistake:** stopping at Step 3. Engineers import the resource, it's in state, and they move on. Then the next `terraform apply` changes or destroys things because the config doesn't match what was imported. **Always drive to zero diff.**

---

### Terraform 1.5+ — Import Blocks (New Way)

```hcl
# You can now declare imports IN config — no CLI command needed
import {
  to = aws_instance.web
  id = "i-0abc1234567890def"
}

resource "aws_instance" "web" {
  ami           = "ami-0f58b397bc5c1f2e8"
  instance_type = "t3.micro"
  # ... fill attributes to match real instance
}
```

```bash
terraform plan    # shows import + any diffs
terraform apply   # executes the import
```

Even better — Terraform 1.5+ can **generate the config for you**:

```bash
terraform plan -generate-config-out=generated.tf
# Writes a .tf file with all attributes from the real resource
# Use this as your starting point, then clean it up
```

This is the fastest way to import complex resources — generate the config, clean up computed/optional attributes, drive to zero diff.

---

## Topic 6.3 — Taint and Force Replace

Sometimes a resource exists and matches config perfectly, but the actual instance is corrupted — needs to be replaced even though Terraform sees no diff.

```bash
# Old way (deprecated but you'll see it in older configs)
terraform taint aws_instance.web
# Marks resource as tainted in state
# Next apply will destroy + recreate it

# New way (Terraform 1.2+)
terraform apply -replace=aws_instance.web
# Does the replace in a single command — no separate taint step
```

**What it looks like in the plan:**

```
# aws_instance.web must be replaced
-/+ resource "aws_instance" "web" {
      # (the instance is marked for replacement by the user)
      ~ id = "i-old" -> (known after apply)
    }
```

**Production use cases:**

```
Corrupted EC2         → app server in bad state, needs fresh AMI boot
Failed config init    → cloud-init failed, instance never configured properly
Security incident     → instance potentially compromised, replace immediately
SSL cert rotation     → old cert cached in memory, force new instance
```

**Important:** `-replace` respects `create_before_destroy` in lifecycle blocks. If you have it set, the new instance comes up before the old one is terminated — zero downtime replacement.

---

## Topic 6.4 — Provider Aliases

For deploying to multiple regions or multiple accounts from one config:

```hcl
# Default provider — ap-south-1
provider "aws" {
  region = "ap-south-1"
}

# Aliased provider — us-east-1
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

# Aliased provider — different account
provider "aws" {
  alias  = "prod_account"
  region = "ap-south-1"

  assume_role {
    role_arn = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformRole"
  }
}
```

**Using aliased providers:**

```hcl
# Uses default provider (ap-south-1)
resource "aws_instance" "web_india" {
  ami           = "ami-india"
  instance_type = "t3.micro"
}

# Explicitly uses aliased provider (us-east-1)
resource "aws_instance" "web_us" {
  provider      = aws.us_east    # reference by alias
  ami           = "ami-us"
  instance_type = "t3.micro"
}

# Uses prod account provider
resource "aws_s3_bucket" "prod_assets" {
  provider = aws.prod_account
  bucket   = "shopsphere-prod-assets"
}
```

**Passing aliased providers into modules:**

```hcl
# Modules need providers passed explicitly when using aliases
module "dr_region" {
  source = "../../modules/compute"

  providers = {
    aws = aws.us_east    # module's "aws" provider = parent's "aws.us_east"
  }

  project     = var.project
  environment = var.environment
}
```

**Production use cases:**

```
Multi-region DR         → primary in ap-south-1, replica in us-east-1
Global resources        → Route53, IAM, CloudFront (us-east-1 only)
Cross-account deploy    → separate AWS accounts for dev/staging/prod
Multi-region S3         → replication source and destination buckets
```

---

## Phase 6 — Provider Execution Flow

Putting it all together — what actually happens when Terraform creates a resource:

```
terraform apply
      │
      ▼
Terraform Core
  reads .tf config
  builds DAG
  determines execution order
      │
      │  for each resource to create:
      ▼
Provider Binary (aws)
  receives: resource type + all attribute values
      │
      ▼
Schema validation
  checks required attributes present
  checks types correct
  checks value constraints
      │
      ▼
AWS API call
  POST https://ec2.ap-south-1.amazonaws.com/
  with SigV4 signed request
      │
      ▼
AWS creates resource
  returns: resource ID + all computed attributes
      │
      ▼
Provider returns to Terraform Core
  all attributes (arguments + computed)
      │
      ▼
Terraform Core writes to state file
  records all attributes
  records dependencies
  increments serial
```

---

## Breaking Exercises

```bash
# Exercise 1 — Import workflow
# Manually create an EC2 instance in AWS console (don't use Terraform)
# Write the resource block in config
# Run terraform import
# Run terraform plan
# Update config until zero diff
# Then manage it going forward with Terraform

# Exercise 2 — Force replace
# Create an EC2 instance via Terraform
# Run terraform apply -replace=aws_instance.web
# Observe: old instance destroyed, new one created
# Add create_before_destroy to lifecycle
# Run -replace again — observe new instance comes up FIRST

# Exercise 3 — Provider debug
# Enable TF_LOG=DEBUG
# Run terraform plan
# Find the actual API calls being made in the logs
# Understand what "describe" calls happen during refresh phase
```

---

## Phase 6 Interview Questions

1. Where does Terraform store provider binaries and what does `terraform init` actually download?
2. How does `terraform import` work? What do you need to do after importing a resource?
3. What is the difference between `terraform taint` and `terraform apply -replace`?
4. When would you use a provider alias? Give a concrete production use case.
5. If a `terraform plan` shows `# forces replacement` on an attribute you barely changed — where do you look to understand why?

---

Say **"move"** for Phase 7 — CI/CD and GitOps, or **"drill"** for the interview questions.
