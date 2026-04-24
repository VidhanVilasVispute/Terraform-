# Phase 2 — Terraform Execution Internals

This is where most engineers have gaps. They use Terraform daily but treat the execution as a black box. That black box will bite you in production.

---

## Topic 2.1 — The Dependency Graph (DAG)

Every resource in your config becomes a **node**. Every reference between resources creates a **directed edge**. Together they form a Directed Acyclic Graph — Terraform walks this to determine execution order.

```
                    aws_vpc.main
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   aws_subnet.pub  aws_subnet.priv  aws_sg.web
          │              │              │
          ▼              │              │
  aws_internet_gw        │              │
          │              │              │
          └──────┬───────┘              │
                 ▼                      │
          aws_nat_gw              aws_instance.web
                 │                      │
                 └──────────────────────┘
                              ▼
                       aws_eip.web
```

Terraform walks this top-down. Resources with no upstream dependencies run **first and in parallel** (up to 10 concurrent). Resources wait only for their direct dependencies.

---

### Implicit vs Explicit Dependencies

**Implicit** — created automatically when you reference another resource's attribute:

```hcl
resource "aws_security_group" "web" { ... }

resource "aws_instance" "web" {
  # This reference IS the dependency declaration
  # Terraform sees it → SG must exist before EC2
  vpc_security_group_ids = [aws_security_group.web.id]
}
```

You don't declare the order — the reference declares it for you.

**Explicit** — `depends_on` for when a real dependency exists but no attribute reference captures it:

```hcl
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy.ec2_policy]
  # IAM policy changes take seconds to propagate globally
  # EC2 launching before propagation = "permission denied" at boot
  # No attribute reference captures this timing dependency
}
```

**Rule:** use `depends_on` only when you have a dependency that doesn't naturally appear as an attribute reference. Overusing it hides relationships and slows down parallelism.

---

### Breaking Exercise

Create a circular dependency on purpose:

```hcl
resource "aws_security_group" "a" {
  ingress {
    security_groups = [aws_security_group.b.id]  # A depends on B
  }
}

resource "aws_security_group" "b" {
  ingress {
    security_groups = [aws_security_group.a.id]  # B depends on A
  }
}
```

Run `terraform plan`. Read the error. Understand it. Then fix it using `aws_security_group_rule` resources instead of inline ingress blocks — that's the real production pattern.

---

## Topic 2.2 — The Plan Engine (The Diff System)

When you run `terraform plan`, three things happen internally in sequence:

```
Step 1 — REFRESH
┌─────────────────────────────────────────────────────┐
│  Terraform calls provider API for each resource     │
│  in state → gets REAL current attributes from AWS   │
│  → updates state with what actually exists NOW      │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
Step 2 — DIFF CALCULATION
┌─────────────────────────────────────────────────────┐
│  Compares desired config (.tf files)                │
│  against refreshed state                            │
│  attribute by attribute, resource by resource       │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
Step 3 — ACTION CLASSIFICATION
┌─────────────────────────────────────────────────────┐
│  + create      → new resource, doesn't exist yet    │
│  ~ update      → change in-place, no replacement    │
│  - destroy     → remove from config                 │
│  -/+ replace   → must destroy + recreate            │
└─────────────────────────────────────────────────────┘
```

---

### Update vs Replace — This Matters

Not all changes are equal. The **provider schema** defines which attributes can be updated in-place and which force a replacement:

```
~ instance_type: "t3.micro" → "t3.medium"
  ✓ AWS supports live instance type change = in-place update

-/+ ami: "ami-old" → "ami-new"  # forces replacement
  ✗ You can't change an AMI on a running instance
    Terraform must destroy and recreate it
```

When you see `-/+` with `# forces replacement` — that's the provider telling you this attribute cannot be changed in place. The provider schema defines this, not Terraform core.

**Production implication:** a `-/+ replace` on an EC2 instance = downtime, unless you have `create_before_destroy = true` in lifecycle. More on that in 2.4.

---

### Breaking Exercise

```bash
# Step 1 — apply an EC2 instance with t3.micro
# Step 2 — change instance_type to t3.small in config
# Step 3 — run terraform plan → observe ~ update

# Step 4 — change the AMI in config
# Step 5 — run terraform plan → observe -/+ forces replacement

# Understand WHY one is update and one is replace
```

---

## Topic 2.3 — Apply Phase Internals

During apply, Terraform walks the DAG in dependency order:

```
aws_vpc.main          ← no dependencies → runs first
        │
        ├── aws_subnet.pub    ┐
        ├── aws_subnet.priv   ├── run in parallel (no deps on each other)
        └── aws_sg.web        ┘
                │
        aws_instance.web      ← waits for both subnet AND sg to finish
                │
        aws_eip.web           ← waits for instance
```

### The Most Important Thing Most Engineers Don't Know

**If apply fails partway through — Terraform does NOT roll back.**

```
Resource 1  ✓ created
Resource 2  ✓ created
Resource 3  ✓ created
Resource 4  ✗ FAILED ← error here
Resource 5    not attempted
Resource 6    not attempted

State file now contains: resources 1, 2, 3
Resources 4, 5, 6: not in state
```

This is intentional — Terraform is not a transaction system. It saves whatever state it has, stops, and reports the error.

**What you do next:** fix the root cause of the failure, then run `terraform apply` again. It will skip resources 1-3 (already exist and match state), retry resource 4, then continue with 5 and 6. This is **idempotent recovery** — the same apply command is safe to re-run.

---

### Breaking Exercise — The Most Important One in This Phase

```bash
# Start a terraform apply
# While it's running → Ctrl+C to kill it midway

# Now do:
cat terraform.tfstate        # what got written?
terraform plan               # what does Terraform propose?
terraform apply              # run again → observe idempotent recovery
```

This teaches you production incident recovery. When an apply fails at 2am, you don't panic and destroy everything. You fix the error and re-apply.

---

## Topic 2.4 — Lifecycle Blocks

These are the knobs that prevent production disasters.

### `create_before_destroy`

```hcl
resource "aws_instance" "web" {
  lifecycle {
    create_before_destroy = true
  }
}
```

**Default behavior** (without this):
```
1. Destroy old EC2   ← DOWNTIME starts here
2. Create new EC2    ← DOWNTIME ends here
```

**With `create_before_destroy`:**
```
1. Create new EC2    ← new instance is running
2. Destroy old EC2   ← old instance removed after new is live
   = zero downtime replacement
```

Use this on any resource where downtime is unacceptable: EC2 instances behind an ALB, launch templates, SSL certificates.

---

### `prevent_destroy`

```hcl
resource "aws_db_instance" "main" {
  lifecycle {
    prevent_destroy = true
  }
}
```

Terraform will **refuse to destroy** this resource even if you tell it to:

```
Error: Instance cannot be destroyed
  Resource aws_db_instance.main has lifecycle.prevent_destroy
  set to true. To allow this object to be destroyed, remove
  this lifecycle argument and re-run Terraform plan/apply.
```

Use on: RDS instances, S3 buckets with production data, anything you cannot accidentally delete.

**Important caveat:** removing `prevent_destroy` from the config and then running apply **will allow the destroy**. It only catches accidental `terraform destroy` or resource removal from config.

---

### `ignore_changes`

```hcl
resource "aws_eks_node_group" "workers" {
  scaling_config {
    desired_size = 3
  }

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size,  # autoscaler changes this at runtime
      tags["LastModified"],            # security scanner adds this
    ]
  }
}
```

Tells Terraform: "I know this attribute will differ from config at runtime — don't treat it as drift."

**Two production use cases:**
- **EKS node groups** — the cluster autoscaler changes `desired_size` dynamically. Without `ignore_changes`, every plan would show a diff and try to reset it.
- **Runtime tags** — security scanners, cost allocation tools, compliance systems add tags automatically. Without `ignore_changes`, Terraform would remove them on every apply.

---

## Phase 2 — Full Execution Flow Summary

```
terraform init
    └── downloads provider binaries into .terraform/

terraform plan
    ├── 1. refresh  → API calls to get real current state
    ├── 2. diff     → config vs refreshed state
    └── 3. output   → classified list of changes

terraform apply
    ├── walks DAG in dependency order
    ├── runs independent resources in parallel (max 10)
    ├── if failure → stops, saves partial state, NO rollback
    └── re-running apply = safe idempotent recovery
```

---

## Phase 2 Interview Questions

These will come up. Answer them cold before moving on:

1. What is the difference between an implicit dependency and `depends_on`? When would you use `depends_on`?
2. A `terraform plan` shows `-/+` next to a resource you only changed a tag on. What do you investigate?
3. A `terraform apply` fails on resource 7 of 12. What is the state of resources 1–6? What do you do next?
4. What does `create_before_destroy` do at the API level? Give a production scenario where it prevents downtime.
5. What does `ignore_changes` do? Name two legitimate production use cases.

---

Say **"move"** for Phase 3 — State Deep Dive, or **"drill"** to work through the interview questions first.
