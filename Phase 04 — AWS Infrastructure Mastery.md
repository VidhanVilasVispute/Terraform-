
# Phase 4 — AWS Infrastructure Mastery

Stop building toy infra. This phase builds something that looks like a real company's network.

---

## Topic 4.1 — Network Layer (VPC)

First, understand the full picture before writing a single resource:

```
                          Internet
                              │
                    ┌─────────────────┐
                    │ Internet Gateway │
                    └─────────────────┘
                              │
               ┌──────────────┴──────────────┐
               │                             │
   ┌───────────────────┐         ┌───────────────────┐
   │  Public Subnet    │         │  Public Subnet    │
   │  AZ-a             │         │  AZ-b             │
   │  10.0.1.0/24      │         │  10.0.2.0/24      │
   │  (ALB, Bastion)   │         │  (ALB multi-AZ)   │
   └───────────────────┘         └───────────────────┘
               │                             │
        NAT Gateway                   NAT Gateway
               │                             │
   ┌───────────────────┐         ┌───────────────────┐
   │  Private Subnet   │         │  Private Subnet   │
   │  AZ-a             │         │  AZ-b             │
   │  10.0.10.0/24     │         │  10.0.20.0/24     │
   │  (App servers)    │         │  (App servers)    │
   └───────────────────┘         └───────────────────┘
```

### Why This Layout

```
Internet Gateway  → allows public subnets to reach internet (bidirectional)
NAT Gateway       → allows private subnets to reach internet (outbound only)
                    private instances can pull updates, call external APIs
                    but cannot be reached FROM the internet directly

Public subnets    → ALB (receives public traffic), Bastion (SSH entry point)
Private subnets   → App servers, databases — no public IPs
```

The core security principle: **app servers are never directly reachable from the internet**. Traffic flows: Internet → IGW → ALB (public subnet) → App server (private subnet).

---

### Building the Network Layer

```hcl
# variables.tf
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "project" {
  type = string
}

variable "environment" {
  type = string
}

# Using cidrsubnet to calculate subnet CIDRs automatically
locals {
  name_prefix = "${var.project}-${var.environment}"

  public_subnet_cidrs  = [
    cidrsubnet(var.vpc_cidr, 8, 1),   # 10.0.1.0/24
    cidrsubnet(var.vpc_cidr, 8, 2),   # 10.0.2.0/24
  ]
  private_subnet_cidrs = [
    cidrsubnet(var.vpc_cidr, 8, 10),  # 10.0.10.0/24
    cidrsubnet(var.vpc_cidr, 8, 20),  # 10.0.20.0/24
  ]
}
```

```hcl
# main.tf — VPC and Subnets

data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${local.name_prefix}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Public subnets — for ALB and bastion
resource "aws_subnet" "public" {
  count = 2

  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true   # instances here get public IPs

  tags = {
    Name = "${local.name_prefix}-public-${count.index + 1}"
    Tier = "public"
  }
}

# Private subnets — for app servers
resource "aws_subnet" "private" {
  count = 2

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  # No map_public_ip_on_launch — private instances get NO public IPs

  tags = {
    Name = "${local.name_prefix}-private-${count.index + 1}"
    Tier = "private"
  }
}
```

---

### Internet Gateway + NAT Gateway

```hcl
# Internet Gateway — public subnets use this to reach internet
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${local.name_prefix}-igw"
  }
}

# Elastic IP for NAT Gateway
# NAT Gateway needs a static public IP
resource "aws_eip" "nat" {
  count  = 2          # one per AZ — high availability
  domain = "vpc"

  depends_on = [aws_internet_gateway.main]
  # EIP association requires IGW to exist first
  # No attribute reference captures this — explicit depends_on needed

  tags = {
    Name = "${local.name_prefix}-nat-eip-${count.index + 1}"
  }
}

# NAT Gateway — sits in PUBLIC subnet, gives private subnets internet access
resource "aws_nat_gateway" "main" {
  count = 2

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  # NAT Gateway lives in PUBLIC subnet — this is correct

  depends_on = [aws_internet_gateway.main]

  tags = {
    Name = "${local.name_prefix}-nat-${count.index + 1}"
  }
}
```

---

### Route Tables — The Critical Part

```hcl
# Public route table — routes internet traffic through IGW
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${local.name_prefix}-public-rt"
  }
}

# Associate public subnets with public route table
resource "aws_route_table_association" "public" {
  count = 2

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private route tables — one per AZ
# Each routes through its AZ's NAT Gateway
resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
    # Routes outbound internet through NAT — NOT through IGW
  }

  tags = {
    Name = "${local.name_prefix}-private-rt-${count.index + 1}"
  }
}

# Associate private subnets with private route tables
resource "aws_route_table_association" "private" {
  count = 2

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

**Why one private route table per AZ:** if AZ-a's NAT Gateway goes down, AZ-b's private subnet still has its own route through AZ-b's NAT Gateway. Single NAT Gateway = single point of failure.

---

## Topic 4.2 — Compute Layer

```hcl
# Bastion host — only entry point for SSH into private network
resource "aws_instance" "bastion" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public[0].id  # PUBLIC subnet
  vpc_security_group_ids      = [aws_security_group.bastion.id]
  associate_public_ip_address = true   # needs public IP — it's the SSH entry point
  key_name                    = var.key_pair_name

  tags = {
    Name = "${local.name_prefix}-bastion"
  }
}

# App server — in PRIVATE subnet, no public IP
resource "aws_instance" "app" {
  count = 2

  ami                         = data.aws_ami.ubuntu.id
  instance_type               = var.instance_type
  subnet_id                   = aws_subnet.private[count.index].id  # PRIVATE
  vpc_security_group_ids      = [aws_security_group.app.id]
  associate_public_ip_address = false  # explicitly no public IP
  iam_instance_profile        = aws_iam_instance_profile.app.name
  key_name                    = var.key_pair_name

  tags = {
    Name = "${local.name_prefix}-app-${count.index + 1}"
  }
}

# Latest Ubuntu 22.04 AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

---

## Topic 4.3 — Security Layer

### Security Groups — The Right Pattern

```hcl
# ALB security group — public facing
resource "aws_security_group" "alb" {
  name        = "${local.name_prefix}-alb-sg"
  description = "ALB — accepts HTTP/HTTPS from internet"
  vpc_id      = aws_vpc.main.id

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

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${local.name_prefix}-alb-sg" }
}

# App security group — only accepts traffic FROM ALB, not from internet
resource "aws_security_group" "app" {
  name        = "${local.name_prefix}-app-sg"
  description = "App servers — only traffic from ALB"
  vpc_id      = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${local.name_prefix}-app-sg" }
}

# Separate rule resource — references ALB SG by ID, not CIDR
# This is the correct production pattern
resource "aws_security_group_rule" "app_from_alb" {
  type                     = "ingress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.alb.id  # SG ID reference
  security_group_id        = aws_security_group.app.id
  description              = "Allow traffic from ALB only"
}

# Bastion security group — SSH from your IP only
resource "aws_security_group" "bastion" {
  name        = "${local.name_prefix}-bastion-sg"
  description = "Bastion — SSH from trusted IPs only"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.trusted_ip_cidr]  # your IP, NOT 0.0.0.0/0
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${local.name_prefix}-bastion-sg" }
}

# App servers accept SSH only from bastion SG
resource "aws_security_group_rule" "app_ssh_from_bastion" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion.id
  security_group_id        = aws_security_group.app.id
  description              = "SSH from bastion only"
}
```

**Why SG ID reference over CIDR:**
```
CIDR-based rule:  allow 10.0.1.0/24 on port 8080
                  → any resource in that subnet can reach app servers
                  → includes things you didn't intend

SG ID reference:  allow traffic FROM resources in aws_security_group.alb
                  → only actual ALB instances, regardless of IP
                  → scales automatically as ALB scales
                  → semantically correct — you mean "from the ALB" not "from that IP range"
```

---

### IAM — Instance Profile Pattern

```hcl
# IAM role — what the EC2 instance IS allowed to do
resource "aws_iam_role" "app" {
  name = "${local.name_prefix}-app-role"

  # Trust policy — who can assume this role
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

# Attach policies — what the role can do
resource "aws_iam_role_policy" "app_s3" {
  name = "${local.name_prefix}-app-s3"
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "arn:aws:s3:::${var.app_bucket}/*"
    }]
  })
}

# Instance profile — the wrapper that attaches a role to EC2
resource "aws_iam_instance_profile" "app" {
  name = "${local.name_prefix}-app-profile"
  role = aws_iam_role.app.name
}

# EC2 uses the profile — NOT hardcoded credentials
resource "aws_instance" "app" {
  iam_instance_profile = aws_iam_instance_profile.app.name
  # App code calls AWS SDK → SDK calls IMDS → gets temporary credentials
  # Credentials rotate automatically, never stored anywhere
}
```

---

### Application Load Balancer

```hcl
resource "aws_lb" "main" {
  name               = "${local.name_prefix}-alb"
  internal           = false   # public facing
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id  # spans BOTH public subnets

  tags = { Name = "${local.name_prefix}-alb" }
}

resource "aws_lb_target_group" "app" {
  name     = "${local.name_prefix}-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    path                = "/actuator/health"  # ShopSphere uses Spring Actuator
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
  }
}

# Register app instances with target group
resource "aws_lb_target_group_attachment" "app" {
  count = 2

  target_group_arn = aws_lb_target_group.app.arn
  target_id        = aws_instance.app[count.index].id
  port             = 8080
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

---

## Phase 4 — Traffic Flow Summary

```
User request
     │
     ▼
Internet Gateway
     │
     ▼
ALB (public subnet, sg: alb-sg)
  accepts: 0.0.0.0/0 on 80/443
     │
     ▼
App Server (private subnet, sg: app-sg)
  accepts: only from alb-sg on 8080
  no public IP — unreachable from internet directly
     │
     ▼ (outbound — package installs, external API calls)
NAT Gateway (public subnet)
     │
     ▼
Internet Gateway → internet
```

---

## Breaking Exercise (Mandatory)

```bash
# Break 1 — Route Table
# Remove the NAT route from the private route table
# Try to install a package on the private EC2 via bastion
# Observe the failure → understand network path debugging

# Break 2 — Security Group
# Change app_from_alb rule to accept from 0.0.0.0/0 instead of ALB SG
# Understand why this is a security problem even if it "works"

# Break 3 — Verify the deliverable
# Private EC2 must be accessible ONLY through ALB
# Prove it:
#   curl http://<ALB_DNS>          → works
#   curl http://<private_IP>       → times out (no route)
#   ssh ec2-user@<private_IP>      → times out (no public IP)
#   ssh bastion → ssh private_EC2  → works (bastion chain)
```

---

## Phase 4 Interview Questions

1. Why should app servers be in private subnets? What is the role of the NAT Gateway?
2. What is the difference between referencing a security group by SG ID vs CIDR in a security group rule? Why does it matter?
3. What is an instance profile? Why should you never hardcode AWS credentials in userdata?
4. If your private EC2 instance can't install packages, what is the most likely network misconfiguration?
5. Why do you need both an Internet Gateway and a NAT Gateway? What does each do?

---

Say **"move"** for Phase 5 — Modules and Project Structure, or **"drill"** for the interview questions.
