# Phase 7 — CI/CD and GitOps

This phase is where Terraform stops being a solo tool and becomes a team practice. Every infrastructure change goes through code review, automated checks, and an approval gate — just like application code.

---

## Topic 7.1 — Why CI/CD for Terraform

Without a pipeline, this is what actually happens on teams:

```
Engineer A                    Engineer B
     │                             │
     ├── terraform plan            │
     │   (looks fine locally)      │
     │                             ├── terraform apply
     │                             │   (changes prod directly)
     ├── terraform apply           │
     │   (conflicts with B's       │
     │    apply — state corrupted) │
```

Problems this causes:
- No audit trail of who changed what and when
- No peer review of infrastructure diffs
- Easy to apply to wrong environment
- No security scanning before changes hit prod
- State corruption from concurrent applies

A proper pipeline solves all of these.

---

## Topic 7.1 — Pipeline Structure

The correct order of operations matters. Each step gates the next.

```
┌─────────────────────────────────────────────────────────────┐
│                    TERRAFORM CI PIPELINE                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. terraform fmt -check                                    │
│     → fails if code isn't formatted                         │
│     → fast, catches style issues before wasting CI time     │
│                                                             │
│  2. terraform validate                                      │
│     → syntax check, type check, reference validation        │
│     → no cloud calls, runs in seconds                       │
│                                                             │
│  3. tfsec . --minimum-severity HIGH                         │
│     → static security analysis                              │
│     → catches open SSH ports, unencrypted buckets, etc.     │
│                                                             │
│  4. conftest test                                           │
│     → policy as code (OPA)                                  │
│     → org-specific rules: no public S3, approved AMIs only  │
│                                                             │
│  5. terraform plan -out=tfplan                              │
│     → generates plan, saves to file                         │
│     → this is what reviewers see                            │
│                                                             │
│  ⏸  MANUAL APPROVAL GATE                                    │
│     → team reviews the plan output                          │
│     → approves or rejects the PR                            │
│                                                             │
│  6. terraform apply tfplan                                  │
│     → applies the SAVED plan file                           │
│     → NOT a fresh plan                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Why apply the saved plan file — not a fresh plan:**

```
Without saved plan:
  plan at 10:00am  → shows change A
  approval at 10:30am
  apply at 10:31am → runs fresh plan → might now show change A + change B
                     you approved A but got A+B

With saved plan:
  plan at 10:00am  → produces tfplan file (binary, tamper-evident)
  approval at 10:30am
  apply tfplan     → executes EXACTLY what was in the plan file
                     what you approved is exactly what runs
```

This is not theoretical — infra changes between plan and apply in real systems. Auto-scaling events, other pipeline runs, manual changes all affect what a fresh plan would produce.

---

### GitHub Actions Implementation

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  pull_request:
    branches: [main]
    paths: ['infra/**']
  push:
    branches: [main]
    paths: ['infra/**']

env:
  TF_VERSION: '1.7.0'
  AWS_REGION: 'ap-south-1'
  WORKING_DIR: 'infra/envs/prod'

jobs:
  validate:
    name: Validate
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ env.WORKING_DIR }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Check formatting
        run: terraform fmt -check -recursive

      - name: Validate config
        run: terraform validate

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1
        with:
          working_directory: ${{ env.WORKING_DIR }}
          minimum_severity: HIGH

  plan:
    name: Plan
    runs-on: ubuntu-latest
    needs: validate               # only runs if validate passes
    if: github.event_name == 'pull_request'
    defaults:
      run:
        working-directory: ${{ env.WORKING_DIR }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        id: plan
        run: terraform plan -out=tfplan -no-color 2>&1 | tee plan_output.txt

      - name: Post plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('${{ env.WORKING_DIR }}/plan_output.txt', 'utf8');
            const truncated = plan.length > 60000
              ? plan.substring(0, 60000) + '\n... [truncated]'
              : plan;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `### Terraform Plan\n\`\`\`hcl\n${truncated}\n\`\`\``
            });

      - name: Upload plan artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: ${{ env.WORKING_DIR }}/tfplan
          retention-days: 5     # plan file expires — cannot apply stale plans

  apply:
    name: Apply
    runs-on: ubuntu-latest
    needs: plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production      # ← this is the manual approval gate
    defaults:
      run:
        working-directory: ${{ env.WORKING_DIR }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Download plan artifact
        uses: actions/download-artifact@v4
        with:
          name: tfplan
          path: ${{ env.WORKING_DIR }}

      - name: Terraform Apply
        run: terraform apply tfplan    # applies SAVED plan, not fresh plan
```

**The `environment: production` block** is how GitHub Actions implements manual approval — you configure required reviewers in the GitHub environment settings. The job pauses until an approved reviewer clicks "Approve deployment."

---

## Topic 7.2 — OIDC Authentication (No Long-Lived Keys)

This is the modern standard. No access keys stored anywhere.

### How It Works

```
GitHub Actions runner
        │
        │  1. requests OIDC token from GitHub
        ▼
GitHub OIDC Provider
        │
        │  2. issues short-lived JWT containing:
        │     - repository name
        │     - branch/ref
        │     - workflow name
        │     - expiry (minutes, not days)
        ▼
AWS STS (AssumeRoleWithWebIdentity)
        │
        │  3. validates JWT signature against GitHub's public keys
        │  4. checks trust policy conditions match
        │  5. issues temporary STS credentials (1 hour max)
        ▼
Terraform runs with temporary credentials
        │
        │  6. credentials expire automatically
        │  7. no rotation needed — new token per job run
```

### AWS Side Setup

```hcl
# Create the OIDC provider in AWS (one-time setup)
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

# IAM role that GitHub Actions will assume
resource "aws_iam_role" "github_actions_terraform" {
  name = "github-actions-terraform"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          # Only YOUR repo on main branch can assume this role
          "token.actions.githubusercontent.com:sub" =
            "repo:VidhanVispute/shopsphere:ref:refs/heads/main"
        }
      }
    }]
  })
}

# Attach Terraform permissions to the role
resource "aws_iam_role_policy_attachment" "github_actions_terraform" {
  role       = aws_iam_role.github_actions_terraform.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
  # In production: scope this down to only what Terraform needs
}
```

**Why this is better than access keys:**

```
Access Keys                      OIDC
───────────────────────────────────────────────────────
Long-lived (months/years)        Expires in minutes
Stored in GitHub Secrets         Never stored anywhere
Must be rotated manually         Rotates automatically per job
If leaked → permanent access     If leaked → already expired
Tied to IAM user                 Tied to specific repo+branch
```

---

## Topic 7.3 — Atlantis (PR-Based GitOps)

Atlantis is a self-hosted bot that runs Terraform operations directly from pull request comments. It's the standard for teams that want full GitOps — every infra change is a PR, reviewed, planned, and applied through the PR lifecycle.

### The Workflow

```
Developer opens PR
        │
        ▼
Atlantis automatically runs:
  terraform fmt -check
  terraform validate
  terraform plan
        │
        ▼
Posts plan output as PR comment:
  ┌─────────────────────────────────────┐
  │  Atlantis Plan for envs/prod        │
  │                                     │
  │  ~ aws_instance.web                 │
  │    instance_type: t3.micro → t3.sm  │
  │                                     │
  │  Plan: 0 to add, 1 to change, 0 to  │
  │  destroy.                           │
  │                                     │
  │  To apply: comment "atlantis apply" │
  └─────────────────────────────────────┘
        │
        ▼
Team reviews plan in PR comments
Approves the PR
        │
        ▼
Authorized user comments "atlantis apply"
        │
        ▼
Atlantis runs terraform apply tfplan
Posts result to PR
        │
        ▼
PR merged to main
```

### Atlantis Config

```yaml
# atlantis.yaml — in repo root
version: 3
automerge: false
projects:
  - name: shopsphere-dev
    dir: infra/envs/dev
    workspace: default
    terraform_version: v1.7.0
    autoplan:
      when_modified: ["../../modules/**/*.tf", "*.tf", "*.tfvars"]
      enabled: true
    apply_requirements:
      - approved          # PR must be approved before apply
      - mergeable         # no merge conflicts

  - name: shopsphere-prod
    dir: infra/envs/prod
    workspace: default
    terraform_version: v1.7.0
    autoplan:
      when_modified: ["../../modules/**/*.tf", "*.tf", "*.tfvars"]
      enabled: true
    apply_requirements:
      - approved
      - mergeable
```

### GitHub Actions vs Atlantis — When to Use Which

```
GitHub Actions                   Atlantis
────────────────────────────────────────────────────────
No extra infra to run            Needs a server (ECS, EC2, k8s)
Approval via environment gates   Approval via PR review + comment
Good for small teams             Better for larger teams
Plan stored as artifact          Plan tied directly to PR
Apply on merge to main           Apply via PR comment before merge
Each env needs separate job      Multi-project config in one file
```

For ShopSphere and your portfolio — GitHub Actions is the right starting point. Atlantis is worth knowing for interviews with larger companies.

---

## Topic 7.4 — Automated Drift Detection

Manual drift checks don't scale. Schedule them.

```yaml
# .github/workflows/drift-detection.yml
name: Drift Detection

on:
  schedule:
    - cron: '0 */6 * * *'   # every 6 hours
  workflow_dispatch:          # also triggerable manually

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]   # check all envs

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_{0}', matrix.environment)] }}
          aws-region: ap-south-1

      - name: Terraform Init
        working-directory: infra/envs/${{ matrix.environment }}
        run: terraform init

      - name: Check for drift
        id: drift
        working-directory: infra/envs/${{ matrix.environment }}
        run: |
          terraform plan -detailed-exitcode -no-color 2>&1 | tee drift_output.txt
          echo "exit_code=${PIPESTATUS[0]}" >> $GITHUB_OUTPUT
        continue-on-error: true    # don't fail job on exit code 2

      - name: Alert on drift
        if: steps.drift.outputs.exit_code == '2'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const output = fs.readFileSync('infra/envs/${{ matrix.environment }}/drift_output.txt', 'utf8');
            // Create GitHub issue, post to Slack, create Jira ticket, etc.
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `[DRIFT] ${{ matrix.environment }} infrastructure drift detected`,
              body: `\`\`\`\n${output.substring(0, 5000)}\n\`\`\``,
              labels: ['infrastructure', 'drift']
            });
```

**Exit code semantics — internalize these:**

```
terraform plan -detailed-exitcode

Exit 0  →  success, no changes        (desired = current)
Exit 1  →  error                      (something broke)
Exit 2  →  success, changes present   (drift detected or config changed)
```

---

## Phase 7 — Full GitOps Flow for ShopSphere

Putting it all together — what the complete flow looks like for a ShopSphere infra change:

```
1. Engineer creates branch: git checkout -b infra/increase-app-instance-type

2. Changes infra/envs/prod/terraform.tfvars:
   instance_type = "t3.small"  # was t3.micro

3. Opens PR → GitHub Actions triggers automatically:
   ✓ fmt check passes
   ✓ validate passes
   ✓ tfsec passes
   ✓ plan runs → posts to PR:
     "~ aws_instance.app[0]: t3.micro → t3.small"
     "~ aws_instance.app[1]: t3.micro → t3.small"
     "Plan: 0 to add, 2 to change, 0 to destroy"

4. Team reviews plan in PR — sees exactly what will change

5. Reviewer approves PR

6. PR merged to main

7. Apply job triggers → pauses at environment gate
   → required reviewer clicks "Approve deployment"

8. terraform apply tfplan runs
   → applies EXACTLY the plan that was reviewed
   → posts apply output to PR (optional)

9. Drift detection job runs 6 hours later
   → confirms no drift → exit 0 → no alert
```

Every change is traceable: who proposed it, what the plan showed, who approved it, when it ran.

---

## Phase 7 Interview Questions

1. How does OIDC-based authentication work with GitHub Actions and AWS? Why is it better than access keys?
2. Why should `terraform apply` always apply a saved plan file rather than running a fresh plan?
3. What is the Atlantis workflow? How does it differ from a standard CI/CD pipeline?
4. How do you implement a manual approval gate in a GitHub Actions pipeline?
5. What is drift detection and how would you implement automated drift alerting?

---

Say **"move"** for Phase 8 — Security and Compliance, or **"drill"** for interview questions.
