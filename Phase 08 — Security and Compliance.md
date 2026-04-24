# Phase 8 — Security and Compliance

Security in Terraform operates at multiple layers. This phase covers all of them — static analysis, secrets management, and policy enforcement.

---

## Topic 8.1 — Static Analysis

Static analysis runs **before** any cloud interaction. It catches misconfigurations in your `.tf` files without needing AWS credentials or running a plan.

### tfsec

```bash
# Install
brew install tfsec          # macOS
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install.sh | bash

# Run
tfsec .                                    # scan current directory
tfsec . --minimum-severity HIGH            # only HIGH and CRITICAL
tfsec . --format json > tfsec_output.json  # machine-readable for CI
```

**What tfsec catches — common findings mapped to ShopSphere:**

```
Finding                              Rule ID        Your config
─────────────────────────────────────────────────────────────────────
S3 bucket public ACL                 aws-s3-001     product images bucket
SG with 0.0.0.0/0 on port 22        aws-ec2-002    bastion misconfiguration
RDS without encryption at rest       aws-rds-001    postgres databases
EC2 without IMDSv2 enforced          aws-ec2-031    all app instances
CloudTrail logging disabled          aws-cloudtrail  audit trail gap
ALB access logs not enabled          aws-elb-001    production ALB
```

**Suppressing false positives:**

```hcl
resource "aws_s3_bucket" "public_assets" {
  bucket = "shopsphere-public-assets"

  #tfsec:ignore:aws-s3-001
  # Reason: this bucket intentionally serves public static assets
  # Approved by: security team, ticket: SEC-123
}
```

Always add a reason comment when suppressing. Unexplained suppressions are a red flag in code review.

---

### checkov

```bash
pip install checkov

checkov -d .                              # scan directory
checkov -d . --framework terraform        # terraform only
checkov -d . --check CKV_AWS_8           # specific check
checkov -d . --skip-check CKV_AWS_21     # skip specific check
checkov -d . --output json               # JSON output for CI
```

**tfsec vs checkov:**

```
tfsec      →  faster, Terraform-specific, simpler rules
checkov    →  slower, multi-framework (Terraform + K8s + Docker + ARM)
               deeper ruleset, better for compliance frameworks
               CIS benchmark, PCI-DSS, SOC2 mappings built in

In practice: run both in CI — they catch different things
```

---

### IMDSv2 — Why It Matters

This comes up in interviews. Understand it properly.

IMDS (Instance Metadata Service) is the endpoint inside every EC2 instance that returns credentials, instance ID, region, and other metadata:

```bash
# Any process on EC2 can call this
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
# Returns: temporary AWS credentials for the instance's IAM role
```

**IMDSv1 problem:** no authentication required. Any process — including SSRF vulnerabilities in your app — can call this endpoint and steal IAM credentials.

**IMDSv2:** requires a session token obtained via a PUT request first. SSRF attacks typically only make GET requests — they can't obtain the session token.

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"   # enforces IMDSv2
    http_put_response_hop_limit = 1            # prevents container escape
  }
}
```

`http_put_response_hop_limit = 1` means the PUT request can only travel one network hop — the instance itself. A container inside the instance cannot reach IMDS with hop limit 1.

tfsec flags `aws-ec2-031` if `http_tokens` is not set to `"required"`.

---

## Topic 8.2 — Secrets Management

Three patterns. Know them in order of quality.

### Pattern 1 — Hardcoded (Never Do This)

```hcl
# WRONG — never commit this
resource "aws_db_instance" "main" {
  password = "mypassword123"   # in git forever, even after deletion
}
```

Even if you delete it from git history, it's in your state file. State files contain every attribute of every resource — including passwords — in plaintext.

---

### Pattern 2 — AWS Secrets Manager (Acceptable)

```hcl
# Store the secret in Secrets Manager first (outside Terraform)
# Then reference it as a data source

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "shopsphere/prod/db-password"
}

resource "aws_db_instance" "main" {
  identifier     = "shopsphere-prod"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.medium"

  username = "shopsphere_app"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string

  # The password is fetched at plan time
  # Never appears in your .tf files or tfvars
  # Still appears in state file — this is the limitation
}
```

**The state file problem with Secrets Manager:**

```json
// terraform.tfstate still contains:
{
  "attributes": {
    "password": "actualpassword123"  ← in state, in plaintext
  }
}
```

This is why:
- State files must be encrypted at rest (S3 SSE)
- State file access must be tightly controlled (S3 bucket policy)
- State files should never be committed to git

---

### Pattern 3 — Vault Dynamic Secrets (Best)

```hcl
terraform {
  required_providers {
    vault = {
      source  = "hashicorp/vault"
      version = "~> 3.0"
    }
  }
}

provider "vault" {
  address = "https://vault.shopsphere.internal"
  # Auth via AWS IAM role — no static Vault token
}

# Vault generates a short-lived credential ON DEMAND
data "vault_aws_access_credentials" "deploy" {
  backend = "aws"
  role    = "shopsphere-terraform-deploy"
  # Vault creates a temporary IAM user or assumes a role
  # Credential expires in minutes/hours
  # No long-lived secret exists anywhere
}

provider "aws" {
  access_key = data.vault_aws_access_credentials.deploy.access_key
  secret_key = data.vault_aws_access_credentials.deploy.secret_key
}
```

**Why dynamic secrets are the best pattern:**

```
Static secret (Secrets Manager)      Dynamic secret (Vault)
──────────────────────────────────────────────────────────────
Created once, used many times         Created per use, expires automatically
Leaked = permanent access             Leaked = already expired
Manual rotation required              No rotation — auto-expires
Appears in state file                 Appears in state but is already stale
Revocation requires knowing the key   Revocation is automatic via TTL
```

**Vault for database credentials (the killer use case):**

```hcl
# Vault generates a unique Postgres username+password per Terraform run
data "vault_database_secret" "postgres" {
  backend = "database"
  name    = "shopsphere-postgres-role"
  # Returns: username=v-token-xyz123, password=A1B2C3...
  # This credential expires in 1 hour
  # No human ever knows what the password is
}

resource "aws_db_instance" "main" {
  username = data.vault_database_secret.postgres.username
  password = data.vault_database_secret.postgres.password
}
```

In state: the credential is there, but it expired an hour ago. An attacker who steals the state file gets credentials that don't work.

---

### The `sensitive` Flag

```hcl
variable "db_password" {
  type      = string
  sensitive = true   # masks value in plan/apply output
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = false
}

output "db_password_out" {
  value     = aws_db_instance.main.password
  sensitive = true   # will show as (sensitive value) in output
}
```

**What `sensitive = true` does:**
```
terraform plan output:  password = (sensitive value)   ← masked
terraform apply output: password = (sensitive value)   ← masked
```

**What `sensitive = true` does NOT do:**
```
state file:   "password": "actualvalue"   ← still plaintext
```

This trips people in interviews. `sensitive = true` is a display mask — it prevents accidental exposure in terminal output and logs. It does not encrypt the value in state.

---

## Topic 8.3 — Policy as Code (OPA + Conftest)

tfsec catches known security anti-patterns. OPA + Conftest enforces **your organization's specific rules** — rules that no generic tool knows about.

Examples of org-specific rules:
- All EC2 instances must use approved AMIs from your internal registry
- All S3 buckets must have a `CostCenter` tag
- No resources may be created in us-west-2 (not an approved region)
- RDS instances must be Multi-AZ in prod

### How It Works

```
terraform plan -out=tfplan
        │
        ▼
terraform show -json tfplan > tfplan.json
        │
        ▼ (tfplan.json is the plan in machine-readable form)
conftest test tfplan.json --policy policy/
        │
        ▼
OPA evaluates your .rego policy files against the plan JSON
Reports: PASS or FAIL with messages
```

### Writing OPA Policies

```rego
# policy/security.rego
package terraform.security

# Deny public S3 buckets
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.after.acl == "public-read"
  msg := sprintf(
    "S3 bucket '%v' must not have public-read ACL",
    [resource.address]
  )
}

# Deny SSH open to the world
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group"
  ingress := resource.change.after.ingress[_]
  ingress.cidr_blocks[_] == "0.0.0.0/0"
  ingress.from_port <= 22
  ingress.to_port >= 22
  msg := sprintf(
    "Security group '%v' must not allow SSH from 0.0.0.0/0",
    [resource.address]
  )
}

# Enforce required tags on all resources
required_tags := {"Environment", "Project", "ManagedBy"}

deny[msg] {
  resource := input.resource_changes[_]
  resource.change.after != null
  existing_tags := {tag | resource.change.after.tags[tag]}
  missing := required_tags - existing_tags
  count(missing) > 0
  msg := sprintf(
    "Resource '%v' is missing required tags: %v",
    [resource.address, missing]
  )
}

# Approved AMI list — only allow images from your internal registry
approved_amis := {
  "ami-0f58b397bc5c1f2e8",   # Ubuntu 22.04 hardened
  "ami-0abc123456789def0",   # Amazon Linux 2023 hardened
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_instance"
  ami := resource.change.after.ami
  not approved_amis[ami]
  msg := sprintf(
    "EC2 instance '%v' uses unapproved AMI '%v'",
    [resource.address, ami]
  )
}
```

### Running Conftest in CI

```bash
# Generate plan JSON
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json

# Run policy checks
conftest test tfplan.json --policy policy/

# Output on violation:
# FAIL - policy/security.rego - data.terraform.security.deny
# S3 bucket 'module.storage.aws_s3_bucket.assets' must not have public-read ACL
# 1 test, 0 passed, 0 warnings, 1 failure

# Exit code 1 on failure — fails the CI pipeline
```

**Where conftest fits in the pipeline:**

```
fmt → validate → tfsec → conftest → plan → approval → apply
                    ↑          ↑
               generic     org-specific
               security     policies
```

tfsec and conftest are complementary — run both. tfsec knows about global AWS security best practices. conftest enforces rules specific to your company.

---

## Phase 8 — Security Layers Summary

```
┌──────────────────────────────────────────────────────────────┐
│                   TERRAFORM SECURITY LAYERS                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Layer 1 — Code (pre-plan)                                   │
│    tfsec      → known security anti-patterns                 │
│    checkov    → compliance framework checks                  │
│    conftest   → org-specific OPA policies                    │
│                                                              │
│  Layer 2 — Plan (post-plan)                                  │
│    conftest against tfplan.json → catches dynamic values     │
│    Human review of plan output                               │
│                                                              │
│  Layer 3 — State                                             │
│    S3 SSE encryption at rest                                 │
│    S3 bucket policy — deny public access                     │
│    DynamoDB encryption                                       │
│    Never commit state to git                                 │
│                                                              │
│  Layer 4 — Secrets                                           │
│    No hardcoded values ever                                  │
│    sensitive = true on variables and outputs                 │
│    Secrets Manager for static secrets                        │
│    Vault for dynamic secrets (production standard)           │
│                                                              │
│  Layer 5 — IAM (runtime)                                     │
│    OIDC — no long-lived keys in CI                           │
│    Instance profiles — no credentials in userdata            │
│    IMDSv2 — prevent SSRF credential theft                    │
│    Least privilege on Terraform execution role               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Breaking Exercises

```bash
# Exercise 1 — tfsec baseline
# Run tfsec against your ShopSphere infra config
# Fix every HIGH and CRITICAL finding
# Add IMDSv2 enforcement to all EC2 instances
# Suppress any legitimate false positives with documented reasons

# Exercise 2 — Write a conftest policy
# Policy: all aws_instance resources must have
#         instance_type from an approved list
# Run: terraform plan -out=tfplan && terraform show -json tfplan > tfplan.json
# Run: conftest test tfplan.json --policy policy/
# Observe the failure, fix the config, observe the pass

# Exercise 3 — Secrets Manager integration
# Store a fake DB password in Secrets Manager manually
# Reference it via data source in your RDS config
# Run terraform plan
# Observe: password not visible in plan output
# Open state file: observe it IS visible there
# Understand why state encryption is non-negotiable
```

---

## Phase 8 Interview Questions

1. What is the difference between tfsec and OPA/Conftest?
2. Why is `sensitive = true` on a variable not sufficient for secret management?
3. Explain how Vault dynamic secrets work and why they're better than static credentials.
4. At what stage of the CI pipeline should policy checks run — before or after plan?
5. What is IMDSv2 and why does it matter for EC2 security?

---

Say **"move"** for Phase 9 — Testing Infrastructure Code, or **"drill"** for interview questions.
