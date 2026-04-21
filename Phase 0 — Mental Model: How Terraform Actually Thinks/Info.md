# Phase 0 — Mental Model: How Terraform Actually Thinks

---

## The Single Most Important Reframe

Most engineers, when they first see Terraform, mentally map it to something familiar:

> *"Oh, it's like a script that runs AWS CLI commands for me."*

That mental model will cause you real pain. Here's why.

When you write a Bash script that calls `aws ec2 run-instances`, you are **telling the machine what to do** — imperative. The script runs top to bottom, executes commands, done.

Terraform is the opposite. You **tell it what the world should look like** — declarative. Terraform figures out how to get there. This is a fundamentally different contract.

---

## The Three Laws of Terraform

Internalize these before anything else.

---

### Law 1 — Desired State vs Current State

At any moment, two worlds exist:

```
┌─────────────────────────────┐     ┌─────────────────────────────┐
│       YOUR .tf FILES        │     │       REAL WORLD (AWS)      │
│                             │     │                             │
│  aws_instance "web" {       │     │  EC2: i-0abc123             │
│    instance_type = t3.micro │     │  type: t3.small  ← DRIFT    │
│    ami           = ami-xyz  │     │  ami:  ami-xyz              │
│  }                          │     │                             │
└─────────────────────────────┘     └─────────────────────────────┘
              │                                   │
              └──────────────┬────────────────────┘
                             ▼
                    Terraform's job:
                    close this gap
```

Your `.tf` files describe **desired state** — what you want to exist.  
The real world is **current state** — what actually exists right now.  
Terraform's entire job is to **close the gap** between the two.

This is why you can run `terraform apply` on already-existing infrastructure and Terraform will say "nothing to do" — because desired state already matches current state.

---

### Law 2 — The State File is Terraform's Memory

Terraform doesn't look directly at AWS to know what it manages. It looks at its **state file** first.

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  .tf config  │         │  State File  │         │  Real AWS    │
│  (desired)   │         │  (believed)  │         │  (actual)    │
└──────────────┘         └──────────────┘         └──────────────┘
        │                       │                        │
        │                       │◄── terraform plan ─────┤
        │                       │    refreshes state      │
        ▼                       ▼                        │
   what you want          what Terraform                 │
                          thinks exists    ◄─────────────┘
                                          then compares
                                          to real world
```

The state file is **Terraform's source of truth about what it has created**. Without it, Terraform has no idea that the EC2 instance with ID `i-0abc123` is the same thing as `aws_instance.web` in your config.

This is why state management is so critical — and why losing the state file is a serious problem.

---

### Law 3 — The Plan is Just a Diff. Nothing Happens Until Apply.

```
terraform plan
```
Produces a diff. Like `git diff`. Nothing changes in the real world.

```
terraform apply
```
Executes the diff. This is when API calls go out.

```
Your config  ──► terraform plan ──►  "here's what I'll do"
                                              │
                              human reviews   │
                                              ▼
                         terraform apply ──► real changes happen
```

This separation is intentional and important. In production, a senior engineer reviews the plan before apply runs. What you're approving is **the exact list of changes** — not a script.

---

## The Execution Flow (Lock This In)

```
Your .tf files
      │
      │  HCL parsing
      ▼
Terraform Core
(builds the DAG —
dependency graph
of all resources)
      │
      │  for each resource, calls
      ▼
Provider Plugin         ← separate binary, downloaded on init
(AWS provider, GCP
provider, etc.)
      │
      │  translates resource blocks
      ▼                into real API calls
Cloud API
(AWS, GCP, Azure...)
      │
      │  records what was created
      ▼
State File
```

**Key insight:** Terraform Core is actually pretty thin. It parses HCL, builds the dependency graph, and orchestrates order. The **provider** does all the real work — it knows what API calls to make, what attributes exist, and which changes require destroying and recreating a resource vs updating in place.

This matters because: if something behaves unexpectedly, the bug is almost always in the **provider**, not Terraform core.

---

## The DAG — Why Order is Automatic

You never write "create the security group first, then the EC2 instance." Terraform figures out order automatically via the **dependency graph**.

```
aws_vpc.main
      │
      ├──► aws_subnet.public
      │           │
      │           └──► aws_instance.bastion
      │
      └──► aws_security_group.app
                  │
                  └──► aws_instance.web ──► aws_eip.web
```

When you write `vpc_security_group_ids = [aws_security_group.app.id]` inside an EC2 resource — that **reference** is an edge in the graph. Terraform sees it and knows: security group must exist before EC2.

Resources with no dependencies on each other run **in parallel**. Terraform doesn't wait for unrelated resources.

---

## The Mental Test

Before we move on, here's a quick mental test. Answer these without looking back:

**1.** You run `terraform apply` on an infrastructure that already exists and perfectly matches your config. What happens?

**2.** You delete a resource block from your `.tf` file and run `terraform plan`. What does Terraform propose?

**3.** Someone goes into the AWS console and manually changes an EC2 instance type. Your `.tf` file still says `t3.micro`. You run `terraform plan`. What happens?

**4.** Why can't you just run `terraform apply` again after someone manually changed something — without looking at the plan first?

