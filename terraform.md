# Terraform

## What is Terraform?

When you provision infrastructure manually — clicking through the AWS console, running CLI commands — it doesn't scale. You can't version it, reproduce it, or hand it to a teammate and get the same result.

**Terraform** is an Infrastructure as Code (IaC) tool — IaC means you describe what infrastructure you want in code files, and Terraform creates, updates, or destroys it to match. The same benefits that code gives you over manual work apply directly here.

> In Ansible: you describe how to configure servers.  
> In Terraform: you describe what infrastructure should exist.

Terraform works with dozens of platforms — AWS, Azure, GCP, Kubernetes — through **providers**.

---

## Why IaC? Why Terraform?

**1. Version control your infrastructure**

Your `.tf` files live in git. You get the full history of every change — who changed what, when, and why. You can review infrastructure changes in a PR just like application code, and roll back if something goes wrong.

**2. Consistent environments**

The same Terraform code that creates your DEV environment creates UAT and PROD. No more "it works in dev but not prod" caused by someone clicking through the console differently. Every environment is a known, reproducible state.

**3. Cost optimisation**

Because spinning up and tearing down is just `terraform apply` and `terraform destroy`, you only pay for what you actually need. Spin up a full environment for testing, destroy it when the test is done. No forgotten instances running over the weekend.

**4. Reusable infrastructure — Modules**

Terraform lets you package infrastructure into **modules** — reusable units you can call with different inputs. Write the VPC setup once, use it across five projects by just changing the variables. Same idea as functions in programming.

**5. Inventory management (CRUD)**

Terraform's state file (`terraform.tfstate`) is a live inventory of everything it manages — what exists, its attributes, its relationships. You always know exactly what's out there. No more spreadsheets or tribal knowledge about what's running where.

**6. Dependency management**

Terraform figures out the order to create resources automatically. If an EC2 instance needs a security group, Terraform creates the security group first — you don't have to think about it. Reference one resource from another (`aws_security_group.name.id`) and Terraform builds the dependency graph for you.

---

## Providers

A **provider** is like a plugin that tells Terraform how to interact with a platform's API.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.48.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

- `terraform {}` block declares which providers your code needs
- `provider "aws" {}` block configures the provider (region, credentials, etc.)
- `terraform init` downloads the provider and creates a lock file (`.terraform.lock.hcl`)

You never call the provider directly — Terraform calls it for you when you run `apply`.

---

## HCL — HashiCorp Configuration Language

Terraform files use **HCL** (HashiCorp Configuration Language). Everything in HCL is a **block**:

```hcl
block_type "label1" "label2" {
  argument = value
}
```

The main block types you'll use:

| Block | Purpose |
|-------|---------|
| `terraform` | Provider requirements and backend config |
| `provider` | Configure a provider (region, credentials) |
| `resource` | Create infrastructure |
| `variable` | Input values (DRY — avoid hardcoding) |
| `output` | Print values after apply |
| `locals` | Intermediate computed values |
| `data` | Read existing infrastructure (not create) |

---

## resource Block

The most common block. Declares a piece of infrastructure to create.

```hcl
resource "aws_instance" "terraform_demo" {
  ami           = "ami-0220d79f3f480ecf5"
  instance_type = "t3.micro"

  tags = {
    Name        = "terraform-demo-1"
    Project     = "roboshop"
    Environment = "dev"
  }
}
```

- `"aws_instance"` — the type of resource (provider\_resourcetype)
- `"terraform_demo"` — your name for it (used to reference it elsewhere in code)
- Arguments inside the block are the resource's configuration

**Reference another resource's attribute:**

```hcl
resource "aws_instance" "terraform_demo" {
  vpc_security_group_ids = [aws_security_group.allow_terraform.id]  # reference SG by name
}
```

`aws_security_group.allow_terraform.id` means: from the resource of type `aws_security_group` named `allow_terraform`, get the `id` attribute. Terraform figures out the dependency order automatically.

---

## Core Commands

```bash
terraform init      # download providers, create lock file
terraform plan      # show what will be created/updated/destroyed
terraform apply     # make it happen (prompts for confirmation)
terraform apply -auto-approve   # skip the confirmation prompt
terraform destroy -auto-approve # destroy all resources
```

**Typical workflow:**

1. Write `.tf` files
2. `terraform init` — sets up the working directory
3. `terraform plan` — review before you act
4. `terraform apply` — create/update resources
5. `terraform destroy` — tear down when done

Terraform stores what it created in `terraform.tfstate`. Never delete this file — it's how Terraform knows what exists.

---

## Variables

Variables make your code DRY — replace hardcoded values with names you can change from outside.

```hcl
variable "instance_type" {
  type    = string
  default = "t3.micro"
}
```

Use a variable with `var.`:

```hcl
resource "aws_instance" "terraform_demo" {
  instance_type = var.instance_type
}
```

### Data types

```hcl
variable "instance_type" {
  type = string        # "t3.micro"
}

variable "port" {
  type = number        # 8080
}

variable "cidr" {
  type = list(string)  # ["0.0.0.0/0"]
}

variable "ec2_tags" {
  type = map           # { Name = "demo", Env = "dev" }
}
```

### Validation

You can reject bad values before Terraform even tries to create anything:

```hcl
variable "instance_type" {
  type = string

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium", "t3.large"], var.instance_type)
    error_message = "Instance type should be either t3.micro or t3.small"
  }
}
```

---

## Variable Precedence (highest → lowest)

You can set a variable's value in four different ways. When the same variable is set in multiple places, this is the winner:

```
1. Command-line      terraform apply -var="instance_type=t3.large"
2. terraform.tfvars  instance_type = "t3.medium"
3. Environment var   export TF_VAR_instance_type=t3.small
4. default           default = "t3.micro"
5. prompt            (Terraform asks if no value is found)
```

**Example:** if you set `default = "t3.micro"`, export `TF_VAR_instance_type=t3.small`, and add `instance_type = "t3.medium"` to `terraform.tfvars`, the command-line `-var` wins. If you don't pass it on CLI, `terraform.tfvars` wins.

### terraform.tfvars

A file Terraform automatically loads to set variable values:

```hcl
# terraform.tfvars
instance_type = "t3.medium"
```

Anything in this file overrides `default` but loses to CLI `-var`.

---

## Conditions (Ternary)

No `if/else` blocks in Terraform — use the ternary expression:

```hcl
expression ? "true-value" : "false-value"
```

Real example — use `t3.micro` for dev, `t3.small` for everything else:

```hcl
resource "aws_instance" "terraform_demo" {
  instance_type = var.environment == "dev" ? "t3.micro" : "t3.small"
}
```

If `var.environment` is `"dev"` → `"t3.micro"`. Any other value → `"t3.small"`.

---

## Loops

### count — list-based loops

`count` creates multiple copies of a resource. The current index is `count.index` (starts at 0).

```hcl
resource "aws_instance" "roboshop" {
  count         = 4
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name = "${var.project}-${var.environment}-${var.instances[count.index]}"
  }
}
```

With `var.instances = ["mongodb", "redis", "mysql", "rabbitmq"]` this creates four EC2 instances named:
- `roboshop-dev-mongodb`
- `roboshop-dev-redis`
- `roboshop-dev-mysql`
- `roboshop-dev-rabbitmq`

`${...}` is **string interpolation** — embed an expression inside a string.

**Reference a specific resource from a count group:**

```hcl
vpc_security_group_ids = [
  aws_security_group.roboshop[count.index].id,  # the SG for this instance
  aws_security_group.common.id                   # a shared SG for all instances
]
```

`aws_security_group.roboshop` is now a list (because it also has `count = 4`), so you access it by index.

### for loop

Transform a list or map into a new list or map:

```hcl
locals {
  upper_instances = [for i in var.instances : upper(i)]
  # ["MONGODB", "REDIS", "MYSQL", "RABBITMQ"]
}
```

### dynamic block

When an argument inside a resource block needs to repeat (like multiple `ingress` rules in a security group), use `dynamic`:

```hcl
resource "aws_security_group" "example" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

`dynamic` generates one `ingress {}` block per item in `var.ingress_rules`. The inner `content {}` block is the template for each repetition.

---

## String Interpolation

Embed expressions inside strings with `${}`:

```hcl
Name = "${var.project}-${var.environment}-${var.instances[count.index]}"
# → "roboshop-dev-mongodb"
```

---

## Real-World Example: Multiple EC2 Instances + Security Groups

```hcl
# variables.tf
variable "instances" {
  type    = list
  default = ["mongodb", "redis", "mysql", "rabbitmq"]
}

variable "project" {
  type    = string
  default = "roboshop"
}

variable "environment" {
  type    = string
  default = "prod"
}
```

```hcl
# main.tf
resource "aws_instance" "roboshop" {
  count         = 4
  ami           = var.ami_id
  instance_type = var.instance_type

  vpc_security_group_ids = [
    aws_security_group.roboshop[count.index].id,
    aws_security_group.common.id
  ]

  tags = {
    Name = "${var.project}-${var.environment}-${var.instances[count.index]}"
  }
}

resource "aws_security_group" "roboshop" {
  count       = 4
  name        = "${var.project}-${var.environment}-${var.instances[count.index]}"
  description = "Allow TLS inbound traffic and all outbound traffic"

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project}-${var.environment}-${var.instances[count.index]}"
  }
}

resource "aws_security_group" "common" {
  name        = "${var.project}-${var.environment}-common"
  description = "Shared security group for all roboshop instances"

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Result: 4 EC2 instances + 4 per-service security groups + 1 common security group, all named consistently.

---

## Quick Reference

| Concept | One-liner |
|---------|-----------|
| Provider | Plugin that tells Terraform how to call a platform's API |
| `terraform init` | Download providers and create lock file |
| `terraform plan` | Preview what will be created/updated/destroyed |
| `terraform apply` | Create or update resources to match your `.tf` files |
| `terraform destroy` | Destroy all resources managed by the state file |
| `terraform.tfstate` | Terraform's record of what it created — never delete |
| `.terraform.lock.hcl` | Locks provider versions — commit this to git |
| HCL | HashiCorp Configuration Language — what `.tf` files are written in |
| Block | Basic unit in HCL — `type "label" { arguments }` |
| `resource` | Declares infrastructure to create |
| `variable` | Input value — makes code reusable and DRY |
| `var.name` | How you reference a variable inside HCL |
| `default` | Fallback value when variable is not set elsewhere |
| `terraform.tfvars` | Auto-loaded file to set variable values |
| `TF_VAR_name` | Environment variable way to set a variable |
| Variable precedence | CLI → tfvars → env var → default → prompt |
| `type = string/number/list/map` | Variable data types |
| `validation` | Block to reject invalid variable values early |
| Ternary | `condition ? "true-val" : "false-val"` — Terraform's only conditional |
| `count` | Create N copies of a resource |
| `count.index` | Zero-based index of the current copy |
| `for` loop | Transform a list/map into another list/map |
| `dynamic` | Generate repeated nested blocks (like multiple ingress rules) |
| String interpolation | `"${var.project}-${var.environment}"` — embed expressions in strings |
| Resource reference | `aws_security_group.name.id` — use another resource's attribute |

---
