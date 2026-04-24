# Phase 3 — State Deep Dive

This is the most important phase. State management is what separates engineers who are dangerous from engineers who just write Terraform.

---

## Topic 3.1 — State File Structure

Before anything else — open a real `.tfstate` file and read it. It's just JSON.

Here's what a real state file looks like after creating an EC2 + Security Group:

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "serial": 12,
  "lineage": "a1b2c3d4-...",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id":            "i-0abc1234567890def",
            "ami":           "ami-0f58b397bc5c1f2e8",
            "instance_type": "t3.micro",
            "public_ip":     "13.233.x.x",
            "private_ip":    "10.0.1.5",
            "arn":           "arn:aws:ec2:ap-south-1:...",
            "tags": {
              "Name":        "shopsphere-dev-web",
              "Environment": "dev"
            }
          },
          "dependencies": [
            "aws_security_group.web"
          ]
        }
      ]
    }
  ]
}
```

### Key Fields

```
version          → state file format version (not your Terraform version)
serial           → increments on every change — if two applies happen
                   simultaneously, the higher serial wins, lower is rejected
lineage          → unique ID for this state file's history
                   prevents accidentally mixing state files from different infra

resources[]      → every resource Terraform manages
  mode           → "managed" (resource) or "data" (data source)
  instances[]    → for count/for_each there are multiple instances here
  attributes     → EVERY attribute as it exists in AWS RIGHT NOW
  dependencies   → other resources this one depends on
```

**The `serial` field is your friend during debugging.** If a CI pipeline fails with a state lock error or write conflict, compare serials — the last successful apply tells you where the good state is.

---

### Resource Address Syntax

Know these cold — you'll use them in every `state` command:

```bash
aws_instance.web                          # single resource
aws_instance.web[0]                       # count-based, index 0
aws_instance.web["prod"]                  # for_each, key "prod"
module.vpc.aws_subnet.public[0]           # resource inside a module
module.eks.module.ng.aws_eks_node_group   # nested modules
```

---

## Topic 3.2 — Remote Backend (S3 + DynamoDB)

Local state is dangerous in a team environment for two reasons:
- Lives only on your machine — teammate has no access
- No locking — two engineers running apply simultaneously = **corrupted state**

### The Standard Production Pattern

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "shopsphere-terraform-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

Each environment gets its own `key` path — completely isolated state files:

```
shopsphere-terraform-state/
├── dev/
│   ├── vpc/terraform.tfstate
│   ├── compute/terraform.tfstate
│   └── rds/terraform.tfstate
├── staging/
│   └── vpc/terraform.tfstate
└── prod/
    ├── vpc/terraform.tfstate
    ├── compute/terraform.tfstate
    └── rds/terraform.tfstate
```

---

### How DynamoDB Locking Actually Works

This is what actually gets written to DynamoDB when a lock is acquired:

```
┌──────────────────────────────────────────────────────────┐
│  DynamoDB Table: terraform-state-lock                    │
├──────────────────────────────────────────────────────────┤
│  LockID    │ shopsphere-state/prod/vpc/terraform.tfstate │
│  Info      │ {                                           │
│            │   "ID": "uuid-here",                        │
│            │   "Operation": "OperationTypeApply",        │
│            │   "Who": "vidhan@hostname",                 │
│            │   "Created": "2024-01-15T10:30:00Z"         │
│            │ }                                           │
└──────────────────────────────────────────────────────────┘
```

**The flow:**

```
Engineer A runs apply
        │
        ├── writes lock record to DynamoDB ──► lock acquired
        │
        │          Engineer B runs apply simultaneously
        │                  │
        │                  ├── tries to write lock
        │                  │
        │                  └── lock already exists → BLOCKED
        │                      "Error: state locked by Engineer A"
        │
        ├── apply completes
        │
        └── deletes lock record ──► lock released
                                         │
                                         └── Engineer B proceeds
```

---

### Recovering a Stale Lock

When a process is killed mid-apply, the lock record stays in DynamoDB forever. Nobody can run plan or apply until you manually delete it:

```bash
# See who holds the lock and when it was created
aws dynamodb get-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "shopsphere-state/prod/vpc/terraform.tfstate"}}'

# Verify the lock is truly stale (check timestamp, check if that process is alive)
# Then delete it
aws dynamodb delete-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "shopsphere-state/prod/vpc/terraform.tfstate"}}'

# Or use Terraform's own command
terraform force-unlock <LOCK_ID>
```

**Never blindly delete a lock.** Check the `Who` and `Created` fields first — if someone is actively applying, deleting their lock causes state corruption.

---

### Setting Up the Backend (Bootstrap Problem)

You cannot use Terraform to create the S3 bucket that Terraform needs for its own state — chicken and egg problem. Bootstrap it manually or with a separate one-time script:

```bash
# Create the state bucket
aws s3api create-bucket \
  --bucket shopsphere-terraform-state \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Enable versioning — critical for state recovery
aws s3api put-bucket-versioning \
  --bucket shopsphere-terraform-state \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket shopsphere-terraform-state \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'

# Create DynamoDB table for locking
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1
```

---

## Topic 3.3 — State Commands

These are your production recovery toolkit. Know every one.

### `terraform state list`

```bash
terraform state list
# aws_instance.web
# aws_security_group.web
# module.vpc.aws_subnet.public[0]
# module.vpc.aws_subnet.public[1]
# module.vpc.aws_vpc.main
```

Your first command in any debugging session — see everything Terraform is tracking.

---

### `terraform state show`

```bash
terraform state show aws_instance.web
# Dumps every single attribute of that resource
# as Terraform knows it in state

# id                    = "i-0abc1234567890def"
# ami                   = "ami-0f58b397bc5c1f2e8"
# instance_type         = "t3.micro"
# private_ip            = "10.0.1.5"
# public_ip             = "13.233.x.x"
# vpc_security_group_ids = ["sg-0xyz"]
```

Use this when you need to know exactly what's in state for a specific resource — before moving, removing, or importing.

---

### `terraform state mv` — The Refactoring Command

This is the most powerful and most important state command.

```bash
terraform state mv aws_instance.web module.compute.aws_instance.web
```

**What it does:** renames a resource address in state **without touching the real infrastructure**.

**Production scenario — safe refactoring:**

You have a flat config and want to wrap resources in a module. Without `state mv`, Terraform would destroy the old resource and create a new one — downtime and data loss.

```bash
# Step 1 — move in state first
terraform state mv aws_instance.web module.compute.aws_instance.web
terraform state mv aws_security_group.web module.compute.aws_security_group.web

# Step 2 — update your config to use the module structure

# Step 3 — run plan
terraform plan
# Should show: No changes. Infrastructure is up-to-date.
# Zero destroy, zero recreate — just a rename in state
```

This is how you safely refactor production infrastructure. ShopSphere context: when you eventually wrap your services in reusable modules, this is how you do it without destroying anything.

---

### `terraform state rm`

```bash
terraform state rm aws_instance.legacy
```

**What it does:** removes a resource from Terraform's state tracking — **does NOT delete the real resource**.

Use cases:
- You want Terraform to "forget" about a resource and stop managing it
- The resource was created outside Terraform and you don't want Terraform touching it
- You're migrating the resource to a different state file

```
Before rm:  Terraform tracks aws_instance.legacy
After rm:   aws_instance.legacy still exists in AWS
            Terraform has no record of it
            Next plan will NOT show it (it's not in state or config)
```

**Critical distinction:**
```
terraform state rm   → forgets the resource, real infra untouched
removing from config → Terraform proposes to destroy it on next plan
```

---

### `terraform state pull`

```bash
terraform state pull
# Outputs the entire state file as JSON to stdout
# Works even with remote backends

terraform state pull > backup.tfstate  # backup before risky operations
```

Always pull and back up state before any manual state surgery.

---

### `terraform import`

Brings an **existing** AWS resource under Terraform management without recreating it:

```bash
terraform import aws_instance.web i-0abc1234567890def
```

**The full import workflow:**

```
Step 1 — Find the real resource ID in AWS
         (EC2: i-xxx, SG: sg-xxx, VPC: vpc-xxx)

Step 2 — Write the resource block in your config
         resource "aws_instance" "web" {
           # attributes will be filled by import
         }

Step 3 — Run terraform import
         terraform import aws_instance.web i-0abc1234567890def

Step 4 — Run terraform plan
         → will show a diff between config and real state

Step 5 — Update config until plan shows zero diff
         This is the critical step most people skip
```

**Why Step 5 matters:** after import, state knows about the resource. But if your config doesn't match what's in state, the next apply will try to change things. You must reconcile your config to match reality.

---

## Topic 3.4 — Drift Detection

Drift is when reality diverges from what Terraform's state says it should be.

```
Terraform state says:    instance_type = t3.micro
Reality in AWS:          instance_type = t3.large   ← someone changed it manually
```

This happens constantly in real companies — engineers make emergency changes in the console, auto-scaling changes resources, external systems modify tags.

### How Terraform Detects Drift

The refresh phase in `terraform plan` catches it:

```bash
terraform plan
# aws_instance.web will be updated in-place:
#   ~ instance_type = "t3.large" -> "t3.micro"
#   # (drift detected: manually changed in console)
```

Terraform is proposing to **bring reality back** to desired state.

### Automated Drift Detection in CI

```bash
# Exit codes from plan:
# 0 = no changes
# 1 = error
# 2 = changes detected (including drift)

terraform plan -detailed-exitcode
if [ $? -eq 2 ]; then
  echo "DRIFT DETECTED — sending alert"
  # post to Slack, create Jira ticket, etc.
fi
```

This is how production teams know when someone has made a manual change — scheduled CI job runs plan every hour, alerts on exit code 2.

---

## Phase 3 — State Mental Model Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    STATE FILE                               │
│                                                             │
│  .tf config  ──►  desired state                            │
│  state file  ──►  what Terraform believes exists           │
│  real AWS    ──►  what actually exists                     │
│                                                             │
│  plan = diff between desired and real (via state refresh)  │
│  apply = closes the gap                                     │
│  drift = real AWS diverged from state                      │
└─────────────────────────────────────────────────────────────┘
```

---

### Breaking Exercises (All Mandatory)

```bash
# Exercise 1 — State Surgery
# Manually edit terraform.tfstate (break something small)
# Use state mv, state rm, import to recover
# Rule: no terraform destroy allowed

# Exercise 2 — Stale Lock
# Start terraform apply
# Kill it with Ctrl+C mid-run
# Observe lock record in DynamoDB
# Manually delete it
# Run apply again — observe clean recovery

# Exercise 3 — Drift Detection
# Create EC2 via Terraform
# Go to AWS console → manually change instance type
# Run terraform plan → observe drift detection
# Decide: do you apply (fix drift) or update config (accept the change)?
```

---

## Phase 3 Interview Questions

1. What is drift? How does Terraform detect it?
2. Explain the S3 + DynamoDB backend pattern. What does DynamoDB actually store in the lock record?
3. A `terraform apply` is killed partway through. What is the state of the state file? What do you do next?
4. What does `terraform state mv` do? Give a real use case where you'd need it.
5. What is the difference between `terraform state rm` and deleting the resource block from config?

---

Say **"move"** for Phase 4 — AWS Infrastructure Mastery, or **"drill"** to work the interview questions.
