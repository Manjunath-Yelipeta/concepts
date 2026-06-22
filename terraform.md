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

### for_each — map-based loops

`count` works on lists and tracks resources by position number. `for_each` works on **maps** and tracks resources by their key name — which makes updates much safer.

The special variable `each` gives you:
- `each.key` — the map key (e.g. `"mongodb"`)
- `each.value` — the map value (e.g. `{ instance_type = "t3.micro" }`)

```hcl
# variables.tf
variable "instances" {
  type = map
  default = {
    mongodb  = { instance_type = "t3.micro" }
    redis    = { instance_type = "t3.micro" }
    mysql    = { instance_type = "t3.micro" }
    frontend = { instance_type = "t3.micro" }
  }
}
```

```hcl
resource "aws_instance" "roboshop" {
  for_each      = var.instances
  ami           = var.ami_id
  instance_type = each.value.instance_type

  tags = {
    Name = "${var.project}-${var.environment}-${each.key}"
    # → "roboshop-dev-mongodb", "roboshop-dev-frontend", etc.
  }
}
```

`aws_instance.roboshop` is now a **map** of resources keyed by component name. Reference an entry by key:

```hcl
vpc_security_group_ids = [
  aws_security_group.roboshop[each.key].id,
  aws_security_group.common.id
]
```

**count vs for_each:**

| | `count` | `for_each` |
|---|---|---|
| Input | list | map |
| Current item | `count.index` (number) | `each.key` / `each.value` |
| Resource address | `aws_instance.x[0]` | `aws_instance.x["mongodb"]` |
| Remove middle item | renumbers everything below it | only removes that one key |

## String Interpolation

Embed expressions inside strings with `${}`:

```hcl
Name = "${var.project}-${var.environment}-${var.instances[count.index]}"
# → "roboshop-dev-mongodb"
```

---

## output Block

`output` prints values after `terraform apply` and exposes them to other modules.

```hcl
output "ec2_instance_output" {
  value = aws_instance.roboshop
}
```

```bash
terraform output                        # show all outputs
terraform output ec2_instance_output    # show a specific output
```

Outputs are also how modules pass values to the code that calls them — the caller reads them as `module.name.output_name`.

---

## Functions

Terraform has many built-in functions. You can't create custom ones — only use what's provided.

### contains — check if an element exists

```hcl
contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
# → true or false

contains(keys(var.instances), "frontend")
# → true if "frontend" is a key in the map
```

Used in variable validation and in conditionals (ternary + count) to check whether something should be created.

### index — find position of an element

```hcl
index(var.instances, "frontend")   # → 9 (zero-based position in the list)
```

### join and split — string ↔ list

```hcl
join(" ", ["Sivakumar", "Reddy", "Mettukuru"])
# → "Sivakumar Reddy Mettukuru"

split(" ", "Sivakumar Reddy Mettukuru")
# → ["Sivakumar", "Reddy", "Mettukuru"]
```

### length — count items

```hcl
length(var.instances)   # number of items in a list or map
```

### keys — get all keys of a map

```hcl
keys(var.instances)
# → ["cart", "catalogue", "frontend", "mongodb", "mysql", ...]
```

### merge — combine maps

Right side wins on duplicate keys:

```hcl
merge(
  { a = "b", c = "d" },
  { c = "z", e = "f" }
)
# → { a = "b", c = "z", e = "f" }
```

**Real pattern — common tags + resource-specific tags:**

Instead of duplicating `Project` and `Environment` in every resource, put shared tags in a variable and merge:

```hcl
variable "common_tags" {
  default = {
    Project     = "roboshop"
    Environment = "dev"
  }
}
```

```hcl
tags = merge(
  var.common_tags,
  {
    Name      = "terraform-demo"
    Component = "catalogue"
  }
)
# → { Project = "roboshop", Environment = "dev", Name = "terraform-demo", Component = "catalogue" }
```

Change `common_tags` once, all resources pick it up.

### lookup — get a value from a map by key

```hcl
lookup(aws_instance.roboshop, "frontend").public_ip
# → the public IP of the frontend instance
```

Useful when iterating over a `for_each` resource group and you need to pull a specific entry by name.

---

## lifecycle

Terraform's default behaviour when a resource attribute changes: **destroy first, then recreate**. This can cause downtime — if a security group is renamed, Terraform tries to delete the old one first, but it's still attached to an EC2 instance, so the deletion fails.

`create_before_destroy` flips the order:

```hcl
resource "aws_security_group" "roboshop" {
  for_each = var.instances
  name     = "${var.project}-${var.environment}-${each.key}"

  lifecycle {
    create_before_destroy = true
  }
}
```

What happens when the SG name changes:
1. New SG created with the new name
2. Instance's SG attachment updated to point at the new SG
3. Old SG destroyed (safely — nothing is attached to it)

Without this, step 3 runs first and fails because the instance still holds a reference.

---

## Dynamic Block

`for_each` on a resource creates multiple resources. `dynamic` is different — it creates multiple **blocks inside** a single resource. The most common use is generating multiple `ingress` rules in a security group without repeating the block.

```hcl
variable "ingress_rules" {
  default = {
    ssh = {
      port        = 22
      cidr_blocks = ["0.0.0.0/0"]
    }
    http = {
      port        = 80
      cidr_blocks = ["0.0.0.0/0"]
    }
    mysql = {
      port        = 3306
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

```hcl
resource "aws_security_group" "allow_terraform" {
  name = "allow_terraform_dynamic"

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

- The label after `dynamic` (`"ingress"`) becomes the special variable inside `content {}` — use `ingress.value` and `ingress.key`
- `content {}` is the template — one copy is generated per map entry
- Adding a new rule only requires adding an entry to `var.ingress_rules`, no new block in the resource

---

## Data Sources

`resource` blocks create things. `data` blocks **read** existing things from the provider without creating or managing them.

Use a data source when something already exists and you just need its information — for example, fetching the latest AMI ID instead of hardcoding it, or reading a VPC that was created outside of Terraform.

```hcl
# data.tf
data "aws_ami" "joindevops" {
  most_recent = true
  owners      = ["973714476881"]   # joindevops AWS account

  filter {
    name   = "name"
    values = ["Redhat-9-DevOps-Practice"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}

output "ami_id" {
  value = data.aws_ami.joindevops.id
}
```

```hcl
# main.tf — use the fetched AMI instead of hardcoding
resource "aws_instance" "terraform_demo" {
  ami           = data.aws_ami.joindevops.id
  instance_type = "t3.micro"
}
```

Reference a data source with `data.<type>.<name>.<attribute>` — same pattern as resources but prefixed with `data.`.

Multiple `filter` blocks narrow the result down. `most_recent = true` picks the latest match when there are multiple AMIs.

---

## State

Terraform's job is to keep **actual infrastructure** matching **desired infrastructure**. The state file is how it tracks what it has created — think of it as Terraform's memory.

```
.tf files       → desired/declared infrastructure
state file      → what Terraform created (its memory)
inside provider → actual infrastructure (what really exists)
```

**How state works across different situations:**

| Situation | State file | Actual infra | What Terraform does |
|-----------|-----------|-------------|-------------------|
| First time (`terraform apply`) | doesn't exist | doesn't exist | creates everything, writes state |
| No changes | matches actual | matches `.tf` files | nothing to do |
| `.tf` files changed | matches actual | differs from `.tf` | updates to match `.tf` |
| Someone changed infra outside Terraform | doesn't match actual | — | detects drift, plans to fix it |
| Resource manually deleted | says it exists | doesn't exist | recreates it |

**Seeing state in action:**

Apply this config — Terraform creates the instance and writes the state file:

```hcl
resource "aws_instance" "terraform_demo" {
  ami           = data.aws_ami.joindevops.id
  instance_type = "t3.micro"
  tags = {
    Name = "terraform-demo-1"
  }
}
```

Now change the `Name` tag and run `terraform plan`:

```hcl
    Name = "terraform-demo-1-change"
```

Terraform compares the state file (says `Name = "terraform-demo-1"`) against your `.tf` files (says `"terraform-demo-1-change"`) and shows:

```
~ aws_instance.terraform_demo
  ~ tags = {
      ~ Name = "terraform-demo-1" -> "terraform-demo-1-change"
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

It knows exactly what changed — because the state file is its memory of what it last created.

**Remote state — storing state in S3:**

In a team, everyone must share the same state file. Store it in S3 and use DynamoDB for locking so two people can't apply at the same time:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }

  backend "s3" {
    bucket       = "my-terraform-state-bucket"
    key          = "roboshop/dev/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true   # native S3 locking — Terraform 1.10+, no DynamoDB needed
  }
}
```

Once configured, `terraform init` migrates the local state to S3. From that point everyone on the team runs against the same state.

**State file security rules:**
- Terraform owns the state file completely — never edit it manually
- Don't give developers permission to delete the state file
- Enable versioning on the S3 bucket (so you can recover old state)
- Enable replication as backup

---

## locals

`locals` are like variables but with extra capabilities:

- You can use other variables inside a local (`var.x`)
- You can use other locals inside a local
- You can assign expressions and functions to them and reuse the result
- You cannot override them from outside (unlike variables) — they are internal to the config

```hcl
# locals.tf
locals {
  name          = "${var.project}-${var.environment}"   # "roboshop-dev"
  instance_type = "t3.micro"
  ami_id        = data.aws_ami.joindevops.id            # pulled from data source

  common_tags = {
    Name        = "${var.project}-${var.environment}"
    Environment = var.environment
    Project     = var.project
  }

  ec2_tags = merge(
    local.common_tags,                                  # local inside local
    { Name = "${local.name}-locals-demo" }              # → "roboshop-dev-locals-demo"
  )

  sg_tags = merge(
    local.common_tags,
    { Name = "${local.name}-common" }                   # → "roboshop-dev-common"
  )
}
```

```hcl
# main.tf — everything comes from locals, nothing hardcoded
resource "aws_instance" "terraform_demo" {
  ami                    = local.ami_id
  instance_type          = local.instance_type
  vpc_security_group_ids = [aws_security_group.allow_terraform.id]
  tags                   = local.ec2_tags
}

resource "aws_security_group" "allow_terraform" {
  name = "${local.name}-common"
  tags = local.sg_tags
}
```

Notice `local.ec2_tags` uses `local.common_tags` inside it — locals can reference other locals. This is something variables cannot do.

**When to use locals vs variables:**

| | `variable` | `local` |
|---|---|---|
| Set from outside | Yes (CLI, tfvars, env) | No — fixed in code |
| Can use expressions/functions | No | Yes |
| Can reference other vars/locals | No | Yes |
| Can reference data sources | No | Yes |
| Use for | inputs from the user | computed intermediate values |

---

## Provisioners

Provisioners run scripts or commands at the time a resource is created or destroyed. They are a last resort — if there is a Terraform resource or an Ansible playbook that can do the job, use that instead.

You can attach multiple provisioners to a single resource. By default they run on **create**. Use `when = destroy` to run one on **destroy** instead.

```hcl
resource "aws_instance" "terraform_demo" {
  ami                    = "ami-0220d79f3f480ecf5"
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.allow_terraform.id]

  # runs on create — write private IP into inventory
  provisioner "local-exec" {
    command = "echo ${self.private_ip} > inventory.ini"
  }

  provisioner "local-exec" {
    command = "echo instance created"
  }

  # runs on destroy — clear the inventory file
  provisioner "local-exec" {
    when    = destroy
    command = "echo instance is going to be destroyed"
  }

  provisioner "local-exec" {
    when    = destroy
    command = "echo > inventory.ini"
  }

  # SSH connection block — required for remote-exec
  connection {
    type     = "ssh"
    user     = "ec2-user"
    password = "DevOps321"
    host     = self.public_ip
  }

  # runs on create — install nginx inside the server
  provisioner "remote-exec" {
    inline = [
      "sudo dnf install nginx -y",
    ]
  }
}
```

`self` refers to the resource being created/destroyed — `self.private_ip`, `self.public_ip`, etc.

**`file`** — copies a file from your local machine into the server (requires a `connection` block):

```hcl
  provisioner "file" {
    source      = "config/app.conf"
    destination = "/etc/app/app.conf"
  }
```

**On-create vs on-destroy summary:**

| | Default | `when = destroy` |
|---|---|---|
| Runs when | `terraform apply` (resource created) | `terraform destroy` (resource deleted) |
| Use for | write inventory, trigger Ansible, notify | clean up inventory, deregister from load balancer |

---

## Real-World Example: Roboshop — EC2 + SG + Route53

All 10 roboshop components, each with its own EC2 instance, its own security group, a private DNS record, and a public DNS record for just the frontend.

```hcl
# variables.tf
variable "instances" {
  type = map
  default = {
    mongodb   = { instance_type = "t3.micro" }
    redis     = { instance_type = "t3.micro" }
    mysql     = { instance_type = "t3.micro" }
    rabbitmq  = { instance_type = "t3.micro" }
    catalogue = { instance_type = "t3.micro" }
    user      = { instance_type = "t3.micro" }
    cart      = { instance_type = "t3.micro" }
    shipping  = { instance_type = "t3.micro" }
    payment   = { instance_type = "t3.micro" }
    frontend  = { instance_type = "t3.micro" }
  }
}

variable "common_tags" {
  default = {
    Project     = "roboshop"
    Environment = "dev"
  }
}
```

```hcl
# main.tf
resource "aws_instance" "roboshop" {
  for_each      = var.instances
  ami           = var.ami_id
  instance_type = each.value.instance_type

  vpc_security_group_ids = [
    aws_security_group.roboshop[each.key].id,
    aws_security_group.common.id
  ]

  tags = merge(var.common_tags, {
    Name      = "${var.project}-${var.environment}-${each.key}"
    Component = each.key
  })
}

resource "aws_security_group" "roboshop" {
  for_each    = var.instances
  name        = "${var.project}-${var.environment}-${each.key}"
  description = "Allow traffic for ${each.key}"

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.common_tags, { Name = "${var.project}-${var.environment}-${each.key}" })

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group" "common" {
  name = "${var.project}-${var.environment}-common"

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(var.common_tags, { Name = "${var.project}-${var.environment}-common" })

  lifecycle {
    create_before_destroy = true
  }
}
```

```hcl
# r53.tf — private DNS record for every component
resource "aws_route53_record" "roboshop" {
  for_each = aws_instance.roboshop
  zone_id  = var.zone_id
  name     = "${each.key}-${var.environment}.${var.domain_name}"
  # → mongodb-dev.daws90s.shop, redis-dev.daws90s.shop, etc.
  type    = "A"
  ttl     = 1
  records = [each.value.private_ip]
}

# public DNS record for frontend only — created conditionally
resource "aws_route53_record" "frontend" {
  count   = contains(keys(var.instances), "frontend") ? 1 : 0
  zone_id = var.zone_id
  name    = "${var.project}-${var.environment}.${var.domain_name}"
  # → roboshop-dev.daws90s.shop
  type    = "A"
  ttl     = 1
  records = [lookup(aws_instance.roboshop, "frontend").public_ip]
}
```

```hcl
# outputs.tf
output "ec2_instance_output" {
  value = aws_instance.roboshop
}
```

What this creates: 10 EC2 instances + 10 per-component SGs + 1 common SG + 10 private Route53 records + 1 public Route53 record for the frontend. All consistently named, all tracked by Terraform state.

---

## Naming Conventions

Consistent naming prevents confusion across environments and teams. Four rules:

1. **Don't repeat the resource type** in the name  
   `roboshop-dev-cart` — good  
   `roboshop-dev-cart-instance` — bad (the resource is already an EC2 instance, no need to say it)

2. **Human-readable names use `-`, internal Terraform names use `_`**  
   Tag `Name = "roboshop-dev-cart"` (hyphen)  
   Resource label `aws_instance.roboshop_cart` (underscore)

3. **Put tags at the end** of the resource block, not inline with other arguments

4. **Plural name for plural resources, singular for single**  
   `aws_subnet.public` (one) vs `var.public_subnet_cidrs` (list of many)

**Resource name pattern used across projects:**

```
<project>-<environment>-<component>[-<az-suffix>]
```

Examples: `roboshop-dev-cart`, `roboshop-prod-public-1a`, `roboshop-dev-nat`

---

## Modules

### Why modules?

The same problem exists in Terraform that exists in any code — copy-pasting infrastructure leads to drift. If you fix a bug in the VPC setup for one project, you want it fixed everywhere. Modules solve this.

| Analogy | Terraform equivalent |
|---------|---------------------|
| `common.sh` in shell scripting | module source file |
| Ansible roles | Terraform modules |
| Functions in Python | module calls in `.tf` |

**Three reasons to write a module:**

1. **Centralised standards** — enforce consistent tagging, naming, security defaults across all projects
2. **Code reuse (DRY)** — write the VPC setup once, call it from every project
3. **Easy maintenance** — fix a bug in the module, all callers pick it up when they upgrade

### Module structure

A module is just a directory of `.tf` files. Conventional layout:

```
terraform-aws-instance/
├── main.tf        # resources
├── variables.tf   # inputs (the module's API)
├── outputs.tf     # outputs (what the caller can read back)
├── locals.tf      # computed intermediate values
└── data.tf        # data sources
```

### Writing a module — terraform-aws-instance

This module wraps a single EC2 instance and always fetches the latest joindevops AMI automatically.

```hcl
# data.tf — always fetches the latest matching AMI; no hardcoded AMI ID
data "aws_ami" "joindevops" {
  most_recent = true
  owners      = ["973714476881"]

  filter { name = "name"              values = ["Redhat-9-DevOps-Practice"] }
  filter { name = "root-device-type"  values = ["ebs"] }
  filter { name = "virtualization-type" values = ["hvm"] }
  filter { name = "architecture"      values = ["x86_64"] }
}
```

```hcl
# variables.tf
variable "project"     { type = string }
variable "environment" { type = string }
variable "component"   { type = string }

variable "instance_type" {
  type    = string
  default = "t3.micro"

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Instance type must be t3.micro, t3.small, or t3.medium"
  }
}

variable "sg_ids"    { type = list }
variable "ec2_tags"  { type = map; default = {} }
```

```hcl
# locals.tf
locals {
  ami_id = data.aws_ami.joindevops.id
  common_tags = {
    Project     = var.project
    Environment = var.environment
    Component   = var.component
  }
}
```

```hcl
# main.tf
resource "aws_instance" "main" {
  ami                    = local.ami_id
  instance_type          = var.instance_type
  vpc_security_group_ids = var.sg_ids

  tags = merge(
    var.ec2_tags,
    local.common_tags,
    { Name = "${var.project}-${var.environment}-${var.component}" }
  )
}
```

```hcl
# outputs.tf
output "public_ip"  { value = aws_instance.main.public_ip }
output "private_ip" { value = aws_instance.main.private_ip }
```

### Writing a module — terraform-aws-vpc

See [terraform-aws-vpc/README.md](../terraform-aws-vpc/README.md) for the full module. Key points:

- Creates VPC, IGW, public/private/database subnets, NAT GW, route tables, and optional VPC peering
- Inputs: `project`, `environment`, subnet CIDRs, `is_peering_required`
- Auto-selects the first two AZs in the region via a `data` source
- Names all resources `<project>-<environment>-<tier>-<az-suffix>` using `locals`

### Calling a module

```hcl
# terraform-ec2-test/main.tf
module "module_test" {
  source = "../terraform-aws-instance"   # path to the module directory

  project       = var.project_name
  environment   = var.env
  component     = var.component_name
  instance_type = var.instance_type
  sg_ids        = var.sg_ids
  ec2_tags      = var.ec2_tags
}
```

```hcl
# vpc-test/main.tf
module "vpc" {
  source              = "../terraform-aws-vpc"
  project             = var.project
  environment         = var.environment
  is_peering_required = true
}
```

### Accessing module outputs

The caller reads module outputs as `module.<name>.<output_name>`:

```hcl
# terraform-ec2-test/outputs.tf
output "pub_ip" {
  value = module.module_test.public_ip   # reads the public_ip output of the module
}

output "private_ip" {
  value = module.module_test.private_ip
}
```

### Module with S3 remote state (Terraform 1.10+)

```hcl
# terraform-ec2-test/provider.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "6.48.0" }
  }

  backend "s3" {
    bucket       = "remote-state-90s"
    key          = "ec2-test.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true   # native S3 state locking — no DynamoDB table needed
  }
}

provider "aws" { region = "us-east-1" }
```

`use_lockfile = true` is the Terraform 1.10+ replacement for `dynamodb_table`. It stores a `.tflock` object in the same S3 bucket — no extra AWS resource to manage.

---

## Multi-Environment Strategies

How do you use the same Terraform code across dev, UAT, and prod? Three approaches, each with trade-offs.

### 1. Workspaces

Terraform workspaces let you run the same code in an isolated state under a different name. The current workspace is available as `terraform.workspace`.

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
```

```hcl
variable "instance_types" {
  default = {
    dev  = "t3.micro"
    prod = "t3.small"
  }
}

resource "aws_instance" "main" {
  instance_type = var.instance_types[terraform.workspace]
}
```

Terraform creates a separate state prefix in S3 for each workspace automatically.

**Advantages:** same code, consistent structure across environments.

**Disadvantages:**
- State is stored in one bucket — no real account-level separation
- Code grows complex: everything is conditioned on `terraform.workspace`
- High risk of accidental changes — you are one `workspace select` away from running against prod
- **Not recommended for production environments**

---

### 2. tfvars per environment

Maintain one codebase with separate `.tfvars` and `backend.tf` files per environment:

```
roboshop-infra/
├── dev/
│   ├── backend.tf      # S3 bucket = roboshop-dev-state
│   └── dev.tfvars      # instance_type = "t3.micro"
└── prod/
    ├── backend.tf      # S3 bucket = roboshop-prod-state
    └── prod.tfvars     # instance_type = "t3.small"
```

```bash
terraform init -reconfigure -backend-config=prod/backend.tf
terraform apply -auto-approve -var-file=prod/prod.tfvars
```

**Advantages:** one codebase, different values per environment.

**Disadvantages:**
- Still easy to accidentally apply dev values to prod by passing the wrong `-var-file`
- No hard boundary between environments at the code level

---

### 3. Separate codebase per environment (recommended)

Each environment gets its own directory and its own S3 bucket:

```
roboshop-infra-dev/    → state in s3://roboshop-dev-state
roboshop-infra-prod/   → state in s3://roboshop-prod-state
```

**Advantages:**
- Zero risk of accidental cross-environment changes
- Clear, hard separation — you can even use separate AWS accounts
- State files are completely isolated

**Disadvantage:** code is duplicated across environment directories.

**How to handle the duplication:** use **modules**. Both `roboshop-infra-dev` and `roboshop-infra-prod` call the same `terraform-aws-vpc` and `terraform-aws-instance` modules with different variable values. The shared logic lives once in the module; the environment directory is just a thin wrapper of variable values and a backend config.

```
workspaces → not recommended (risky, state not isolated)
tfvars     → acceptable, but still risky in practice
separate codebase + modules → recommended (safe separation, DRY via modules)
```

---

## VPC on AWS — Concepts

### What is a VPC?

A **Virtual Private Cloud** is a logically isolated section of the AWS cloud — like a private data center that only you can access. All resources for a project live inside a VPC; nothing outside can reach them unless you explicitly allow it.

### IP Addressing and CIDR

Every device on a network needs an IP address. IPv4 addresses are 32 bits written as four 8-bit octets:

```
192.168.1.5

Binary: 11000000.10101000.00000001.00000101
        192      .168      .1        .5
```

**CIDR (Classless Inter-Domain Routing)** notation specifies a range of IP addresses.  
`/N` means the first N bits are fixed (the network part); the remaining bits are free (host addresses).

```
10.0.0.0/16  →  first 16 bits fixed (10.0), last 16 bits free
                 2^16 = 65,536 addresses: 10.0.0.0 → 10.0.255.255

10.0.1.0/24  →  first 24 bits fixed (10.0.1), last 8 bits free
                 2^8 = 256 addresses: 10.0.1.0 → 10.0.1.255
```

### Subnets

A subnet is a logical partition of the VPC CIDR. You divide a `/16` VPC into smaller `/24` subnets to separate tiers of your application:

```
VPC: 10.0.0.0/16

Public subnets:   10.0.1.0/24  (1a),  10.0.2.0/24  (1b)
Private subnets:  10.0.11.0/24 (1a),  10.0.12.0/24 (1b)
Database subnets: 10.0.21.0/24 (1a),  10.0.22.0/24 (1b)
```

**Why three tiers?**

| Tier | Who can reach it | Contains |
|------|-----------------|---------|
| Public | Internet + VPC | Load balancers, NAT Gateway, bastion |
| Private | VPC only (via NAT for outbound) | Application servers |
| Database | VPC only (via NAT for outbound) | RDS, ElastiCache |

**Always span at least 2 Availability Zones** — if one AZ fails, the other keeps your service running (High Availability).

### Internet Gateway

An Internet Gateway is attached to the VPC and enables two-way communication between the VPC and the internet — like the router/modem at the edge of the network.

Subnets with a route pointing to the IGW are **public subnets**. Subnets without that route are **private**.

### Route Tables

Route tables are the traffic rules — they tell packets where to go.

```
Public route table:
  10.0.0.0/16  → local (stay inside VPC)
  0.0.0.0/0    → Internet Gateway   ← this is what makes it "public"

Private route table:
  10.0.0.0/16  → local
  0.0.0.0/0    → NAT Gateway        ← outbound only; no inbound from internet

Database route table:
  10.0.0.0/16  → local
  0.0.0.0/0    → NAT Gateway
```

### NAT Gateway

Private and database instances need outbound internet access (to install packages, download updates) but must not be reachable from the internet.

A **NAT Gateway** sits in a public subnet with an Elastic IP. Private instances route their outbound traffic through it — the NAT Gateway forwards the request to the internet and returns the response. Inbound connections from the internet cannot be initiated.

```
Private instance → NAT Gateway (public subnet, EIP) → Internet
Internet → [blocked — cannot initiate connection to private instance]
```

### VPC Peering

Two VPCs can communicate directly without going through the internet using **VPC peering**. Common use case: a `dev` VPC needing to reach a shared services VPC (like the AWS default VPC where tools run).

```hcl
resource "aws_vpc_peering_connection" "default" {
  count       = var.is_peering_required ? 1 : 0
  vpc_id      = aws_vpc.main.id          # requester
  peer_vpc_id = data.aws_vpc.default.id  # accepter (default VPC)
  auto_accept = true

  accepter  { allow_remote_vpc_dns_resolution = true }
  requester { allow_remote_vpc_dns_resolution = true }
}
```

### Complete VPC in Terraform — What gets created

```
aws_vpc
aws_internet_gateway         → attached to VPC
aws_subnet.public[0,1]       → map_public_ip_on_launch = true
aws_subnet.private[0,1]
aws_subnet.database[0,1]
aws_route_table.public       → route: 0.0.0.0/0 → IGW
aws_route_table.private      → route: 0.0.0.0/0 → NAT GW
aws_route_table.database     → route: 0.0.0.0/0 → NAT GW
aws_route_table_association  × 6 (one per subnet)
aws_eip.nat                  → Elastic IP for NAT GW
aws_nat_gateway              → placed in public subnet[0]
aws_route (public/private/database)
aws_vpc_peering_connection   → optional
```

The `terraform-aws-vpc` module encapsulates all of the above. Callers only need:

```hcl
module "vpc" {
  source              = "../terraform-aws-vpc"
  project             = "roboshop"
  environment         = "dev"
  is_peering_required = true
}
```

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
| `count` | Create N copies of a resource; tracks by position number |
| `count.index` | Zero-based index of the current copy |
| `for_each` | Create one resource per map entry; tracks by key name — safer than count |
| `each.key` / `each.value` | Current map key and value inside a for_each resource |
| `dynamic` | Generate repeated nested blocks inside a resource (e.g. multiple ingress rules) |
| `content {}` | Template block inside `dynamic` — one copy generated per iteration |
| String interpolation | `"${var.project}-${var.environment}"` — embed expressions in strings |
| Resource reference | `aws_security_group.name.id` — use another resource's attribute |
| `locals` | Internal computed values — can use vars, other locals, and functions; not overridable from outside |
| `local.name` | How you reference a local value |
| State file | Terraform's memory — records what it created; never edit manually |
| Remote state | Store state in S3 so the whole team shares it; lock it to prevent concurrent applies |
| `data` block | Read existing infrastructure from the provider without creating it |
| `data.<type>.<name>.<attr>` | How you reference a data source attribute |
| `local-exec` provisioner | Run a command on the machine running Terraform |
| `remote-exec` provisioner | Run commands inside the newly created server over SSH |
| `file` provisioner | Copy a file from local machine into the server |
| `self` | Inside a provisioner, refers to the resource being created |
| `output` | Print a value after apply; expose values from modules |
| `contains(list, val)` | Check if an element exists in a list — true/false |
| `index(list, val)` | Find the position of an element in a list |
| `join(sep, list)` | Combine a list into a string with a separator |
| `split(sep, str)` | Split a string into a list |
| `length(list/map)` | Count items in a list or map |
| `keys(map)` | Get all keys of a map as a list |
| `merge(map1, map2)` | Combine two maps; right side wins on duplicate keys |
| `lookup(map, key)` | Get a value from a map by key |
| `common_tags` pattern | Put shared tags in a variable, use `merge()` to add resource-specific ones |
| `lifecycle` | Block that overrides Terraform's default resource update behaviour |
| `create_before_destroy` | Create the replacement first, then destroy the old — prevents downtime |
| **Naming convention** | `<project>-<environment>-<component>[-<az>]`; hyphens in names, underscores in Terraform labels |
| **Module** | Reusable directory of `.tf` files; inputs via `variables.tf`, outputs via `outputs.tf` |
| `module "x" { source = "..." }` | Call a module; pass values as arguments, read back with `module.x.output_name` |
| `use_lockfile = true` | Terraform 1.10+ S3 native state locking — no DynamoDB table required |
| **Workspaces** | Multiple envs from one codebase via `terraform.workspace`; risky, not recommended for prod |
| **tfvars per env** | Separate `.tfvars` + backend config per env; lower risk than workspaces but still error-prone |
| **Separate codebase + modules** | One directory per env with its own S3 bucket; safest, use modules to avoid code duplication |
| **VPC** | Isolated private network in AWS — like a data center you control |
| **CIDR /N** | First N bits fixed (network), remaining bits free (hosts); `/16` = 65 536 IPs, `/24` = 256 IPs |
| **Public subnet** | Has a route to the Internet Gateway — instances can receive inbound traffic from internet |
| **Private subnet** | No inbound from internet; outbound via NAT Gateway only |
| **Internet Gateway** | VPC-level router to the internet — attach one per VPC |
| **NAT Gateway** | In a public subnet with EIP; lets private instances reach internet outbound-only |
| **VPC Peering** | Direct private link between two VPCs without internet; `auto_accept = true` for same-account |
| **Route table** | Traffic rules — `0.0.0.0/0 → IGW` for public, `0.0.0.0/0 → NAT GW` for private/database |

---
