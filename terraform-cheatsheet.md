
# Terraform Cheatsheet (Commands, Codes & Steps)

> A fast, practical reference for everyday Terraform work—installation, init → plan → apply workflow, variables & modules, state, workspaces, backends, and common HCL patterns.

---

## 0) Install & Setup

### macOS (Homebrew)
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform -install-autocomplete
terraform version
```

### Linux (Zip release)
```bash
curl -fsSL https://releases.hashicorp.com/terraform/ | head -n 20  # find latest version
# Example for a specific version (adjust for your OS/arch):
curl -O https://releases.hashicorp.com/terraform/1.7.5/terraform_1.7.5_linux_amd64.zip
unzip terraform_1.7.5_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform -install-autocomplete
terraform version
```

### Optional: tfenv (version manager)
```bash
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bash_profile
source ~/.bash_profile
tfenv install 1.7.5
tfenv use 1.7.5
```

---

## 1) Project Scaffolding

```
project/
├─ main.tf          # root module (resources, data, providers)
├─ variables.tf     # input variables
├─ outputs.tf       # outputs
├─ providers.tf     # required_providers + provider blocks
├─ versions.tf      # required_version, etc.
├─ terraform.tfvars # default variable values (git-ignore secrets)
└─ modules/         # reusable child modules
```

Recommended .gitignore:
```
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.*
crash.log
*.tfvars
*.auto.tfvars
override.tf
override.tf.json
*_override.tf
*_override.tf.json
```

---

## 2) Initialize Providers, Modules & Backend

```bash
terraform init                    # first command in any new or cloned repo
terraform init -upgrade           # update providers/modules to latest allowed
terraform providers               # show required providers
terraform -help <command>         # built-in help
```

**Migrate / reconfigure backend**
```bash
# supply backend config file(s) during init
terraform init -backend-config=backend.hcl -reconfigure
# or migrate state to a different backend
terraform init -migrate-state -force-copy
```

Example `backend.hcl` for AWS S3 remote state with DynamoDB locking:
```hcl
bucket         = "my-tf-state-bucket"
key            = "envs/dev/terraform.tfstate"
region         = "ap-south-1"
dynamodb_table = "tf-state-locks"
encrypt        = true
```
And in `providers.tf`:
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {}
}

provider "aws" {
  region = var.aws_region
}
```

---

## 3) The Core Workflow

```bash
terraform fmt -recursive           # format HCL
terraform validate                 # static validation
terraform plan -out=plan.tfplan    # create plan
terraform apply plan.tfplan        # apply the saved plan
# or directly:
terraform apply                    # will ask for approval
terraform apply -auto-approve      # non-interactive
terraform destroy                  # tear everything down
```

**Targeted actions (use sparingly)**
```bash
terraform plan   -target=aws_s3_bucket.site
terraform apply  -target=module.network
terraform destroy -target=aws_iam_role.example
```

**Replace a specific resource on next apply**
```bash
terraform plan   -replace=aws_instance.web
terraform apply  -replace=aws_instance.web
```

**Troubleshooting**
```bash
TF_LOG=DEBUG TF_LOG_PATH=tf-debug.log terraform apply
terraform console                         # try expressions against current state
terraform graph | dot -Tpng > graph.png   # visualize dependencies (Graphviz)
```

---

## 4) Variables, Locals & Outputs

`variables.tf`:
```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default     = {
    project = "cloudnautic-demo"
    owner   = "Atul"
  }
}
```

`terraform.tfvars` (auto-loaded):
```hcl
aws_region = "ap-south-1"
tags = {
  project = "cloudnautic-demo"
  owner   = "Atul"
}
```

Passing variables:
```bash
terraform apply -var 'aws_region=ap-south-1'
terraform apply -var-file=prod.tfvars
```

`locals` & string interpolation:
```hcl
locals {
  name_prefix = "${var.project}-${terraform.workspace}"
  common_tags = merge(var.tags, { workspace = terraform.workspace })
}
```

`outputs.tf`:
```hcl
output "website_url" {
  value = "http://${aws_s3_bucket_website_configuration.site.website_endpoint}"
}
# CLI
# terraform output
# terraform output -raw website_url
# terraform output -json | jq
```

---

## 5) Resources, Data Sources & Meta-Arguments

**Resource:**
```hcl
resource "aws_s3_bucket" "site" {
  bucket = "${local.name_prefix}-site"
  tags   = local.common_tags
}
```

**Data source:**
```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}
```

**Meta-arguments:**
```hcl
# count
resource "aws_iam_user" "demo" {
  count = 3
  name  = "demo-${count.index}"
}

# for_each
resource "aws_s3_bucket" "b" {
  for_each = toset(["logs","assets","backup"])
  bucket   = "${local.name_prefix}-${each.key}"
}

# depends_on
resource "aws_iam_user_policy_attachment" "attach" {
  for_each  = toset(["ReadOnlyAccess"])
  user      = aws_iam_user.demo[0].name
  policy_arn = "arn:aws:iam::aws:policy/${each.key}"
  depends_on = [aws_iam_user.demo]
}

# lifecycle
resource "aws_s3_bucket" "immutable" {
  bucket = "${local.name_prefix}-immutable"
  lifecycle {
    prevent_destroy = true
    ignore_changes  = [tags]
  }
}
```

**Conditionals & splats:**
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.enable_large ? "t3.large" : "t3.micro"
  tags          = { Name = local.name_prefix }
}

output "instance_ids" {
  value = aws_instance.web[*].id  # full splat list
}
```

---

## 6) Modules (Re-use & Composition)

`modules/vpc/main.tf`:
```hcl
variable "cidr" { type = string }
resource "aws_vpc" "this" { cidr_block = var.cidr }
output "id" { value = aws_vpc.this.id }
```

Use the module from root:
```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

module "web" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "~> 5.0"
  name    = "${local.name_prefix}-web"
  ami     = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = module.vpc.public_subnets[0]
  tags          = local.common_tags
}
```

**Get / update modules**
```bash
terraform get -update=true
```

---

## 7) Workspaces (env-style isolation)

```bash
terraform workspace list
terraform workspace new dev
terraform workspace select dev
terraform workspace show
```

Use `terraform.workspace` in code to suffix names per environment.

---

## 8) State: Inspect, Move, Import

```bash
terraform show                    # human-readable state
terraform state list              # all addresses
terraform state show aws_s3_bucket.site
terraform state mv aws_iam_role.old module.iam.aws_iam_role.new
terraform state rm aws_s3_bucket.legacy
terraform state replace-provider registry.terraform.io/hashicorp/aws                                   registry.opentofu.org/hashicorp/aws
terraform state pull  > terraform.tfstate
terraform state push  terraform.tfstate
```

**Import existing resources (define resource in code first)**
```bash
# In code:
resource "aws_s3_bucket" "imported" { bucket = "my-existing-bucket" }

# Then import by ID (varies per type)
terraform import aws_s3_bucket.imported my-existing-bucket
```

---

## 9) Terraform Cloud (Remote runs & state)

```bash
terraform login             # authenticate to TFC/Enterprise
terraform logout
```

In `terraform` block, configure `cloud {}` or `backend "remote" {}` as needed.

---

## 10) Examples: Minimal AWS Static Website

`variables.tf`:
```hcl
variable "aws_region" { type = string, default = "ap-south-1" }
```

`main.tf`:
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {}
}

provider "aws" { region = var.aws_region }

locals {
  name = "tf-site-${terraform.workspace}"
}

resource "aws_s3_bucket" "site" {
  bucket = local.name
  tags   = { Project = "cloudnautic", Env = terraform.workspace }
}

resource "aws_s3_bucket_website_configuration" "site" {
  bucket = aws_s3_bucket.site.id
  index_document { suffix = "index.html" }
  error_document { key = "index.html" }
}

output "url" {
  value = "http://${aws_s3_bucket_website_configuration.site.website_endpoint}"
}
```

Commands:
```bash
terraform init -backend-config=backend.hcl     # setup state bucket/lock table
terraform workspace new dev && terraform workspace select dev
terraform plan -out=plan.tfplan
terraform apply plan.tfplan
terraform output url
```

---

## 11) Tips & Good Practices

- Keep provider versions pinned (`~>`), commit `.terraform.lock.hcl`.
- Never commit `*.tfstate` or secrets (`*.tfvars` with credentials).
- Use `for_each` over `count` for stable addressing.
- Prefer modules for repeatable stacks; keep root thin.
- Minimize `-target`; it bypasses dependency graph and can cause drift.
- Use `-refresh-only` plans to inspect drift without writing state:
  ```bash
  terraform plan -refresh-only
  ```
- Break down large applies with `-parallelism=N` if provider/API throttles.
- CI/CD: run `fmt`, `validate`, `init`, `plan`, capture plan as artifact; require manual approval before `apply` (or use Terraform Cloud workspaces).

---

## 12) One-Page Command Reference (copy/paste)

```bash
terraform version
terraform init [-backend-config=FILE] [-upgrade] [-reconfigure]
terraform fmt -recursive
terraform validate
terraform plan [-out=plan.tfplan] [-target=ADDR] [-replace=ADDR] [-refresh-only]
terraform apply [plan.tfplan] [-auto-approve]
terraform destroy [-target=ADDR] [-auto-approve]
terraform output [-raw NAME] [-json]
terraform providers
terraform console
terraform graph | dot -Tpng > graph.png
terraform workspace list|new|select|show
terraform state list|show|mv|rm|pull|push|replace-provider
terraform import <ADDR> <ID>
```

---

**© 2025 — Compact guide prepared for Atul (Cloud Solutions Architect).**


---

## Terraform Quick Reference (Extended Commands)

### Terraform Overview
Terraform is a tool for building, changing, and versioning infrastructure safely and efficiently. Terraform can manage existing and popular service providers as well as custom in-house solutions.

👉 [Terraform-Commands-Cheatsheet](https://atulkamble.github.io/Terraform-Commands-Cheatsheet/)

_No need to run in terror from Terraform. Close that search engine tab and check out the ultimate Terraform Cheatsheet by [Atul Kamble](https://atulkamble.github.io/)._

---

### Terraform Command Lines

#### CLI Tricks
```bash
terraform -install-autocomplete   # Setup tab auto-completion, requires logging back in
```

#### Format and Validate Terraform Code
```bash
terraform fmt                                # format code per HCL canonical standard
terraform validate                           # validate code for syntax
terraform validate -backend=false            # skip backend validation
```

#### Initialize Working Directory
```bash
terraform init                               # initialize directory, pull providers
terraform init -get-plugins=false            # init without downloading plugins
terraform init -verify-plugins=false         # init without verifying plugin signatures
```

#### Plan, Deploy & Cleanup Infrastructure
```bash
terraform apply --auto-approve               # apply without prompt
terraform destroy --auto-approve             # destroy without prompt
terraform plan -out plan.out                 # save plan to file
terraform apply plan.out                     # apply saved plan
terraform plan -destroy                      # output destroy plan
terraform apply -target=aws_instance.my_ec2  # apply only target resource
terraform apply -var my_region=us-east-1     # pass variable from CLI
terraform apply -lock=true                   # lock state file (if backend supports)
terraform apply -refresh=false               # skip refresh step (faster for large infra)
terraform apply --parallelism=5              # limit concurrent ops
terraform refresh                            # reconcile state with real resources
terraform providers                          # show providers in use
```

#### Workspaces
```bash
terraform workspace new mynewworkspace
terraform workspace select default
terraform workspace list
```

#### State Manipulation
```bash
terraform state show aws_instance.my_ec2
terraform state pull > terraform.tfstate
terraform state mv aws_iam_role.my_ssm_role module.custom_module
terraform state replace-provider hashicorp/aws registry.custom.com/aws
terraform state list
terraform state rm aws_instance.myinstance
```

#### Import & Outputs
```bash
terraform import aws_instance.new_ec2 i-abcd1234
terraform import 'aws_instance.new_ec2[0]' i-abcd1234
terraform output
terraform output instance_public_ip
terraform output -json
```

#### Miscellaneous
```bash
terraform version
terraform get -update=true
```

#### Console (test interpolations)
```bash
echo 'join(",",["foo","bar"])' | terraform console
echo '1 + 5' | terraform console
echo "aws_instance.my_ec2.public_ip" | terraform console
```

#### Graph (dependencies)
```bash
terraform graph | dot -Tpng > graph.png
```

#### Taint & Unlock
```bash
terraform taint aws_instance.my_ec2
terraform untaint aws_instance.my_ec2
terraform force-unlock LOCK_ID
```

#### Terraform Cloud
```bash
terraform login
terraform logout
```

---

## Author Profile (Study Guide + Notes)
- [LinkedIn: atuljkamble](https://www.linkedin.com/in/atuljkamble)
- [Twitter: atul_kamble](https://www.twitter.com/atul_kamble)
- [GitHub: atulkamble](https://www.github.com/atulkamble)
- [Medium: atuljkamble](https://atuljkamble.medium.com/)
- [HashNode: atulkamble](https://hashnode.com/@atulkamble)
- [Dev: atulkamble](https://dev.to/atulkamble)
- [Twitch: atulkamble](https://www.twitch.tv/atulkamble)
- [Kaggle: atuljkamble](https://www.kaggle.com/atuljkamble)
- [GitLab: atulkamble](https://gitlab.com/atulkamble)
