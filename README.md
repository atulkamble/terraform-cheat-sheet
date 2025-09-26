# Terraform Cheatsheet (Commands, Codes & Steps)

> A fast, practical reference for everyday Terraform work — installation, init → plan → apply workflow, variables & modules, state, workspaces, backends, and common HCL patterns.

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
curl -O https://releases.hashicorp.com/terraform/1.7.5/terraform_1.7.5_linux_amd64.zip
unzip terraform_1.7.5_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform -install-autocomplete
terraform version
```

### Optional: tfenv (Version Manager)

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
├─ providers.tf     # provider blocks
├─ versions.tf      # required_version, etc.
├─ terraform.tfvars # variable defaults (don’t commit secrets)
└─ modules/         # reusable child modules
```

Recommended `.gitignore`:

```
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.*
crash.log
*.tfvars
*.auto.tfvars
override.tf*
*_override.tf*
```

---

## 2) Initialize Providers, Modules & Backend

```bash
terraform init                    # first command in a new repo
terraform init -upgrade           # upgrade providers/modules
terraform init -backend-config=backend.hcl -reconfigure
terraform providers               # list required providers
terraform -help <command>         # built-in help
```

Example `backend.hcl` (AWS S3 + DynamoDB):

```hcl
bucket         = "my-tf-state-bucket"
key            = "envs/dev/terraform.tfstate"
region         = "ap-south-1"
dynamodb_table = "tf-state-locks"
encrypt        = true
```

---

## 3) Core Workflow

```bash
terraform fmt -recursive                 # format
terraform validate                       # validate HCL
terraform plan -out=plan.tfplan          # generate plan
terraform apply plan.tfplan              # apply saved plan
terraform apply [-auto-approve]          # direct apply
terraform destroy [-auto-approve]        # destroy infra
```

Targeted & replace:

```bash
terraform plan -target=aws_s3_bucket.site
terraform apply -replace=aws_instance.web
```

Troubleshooting:

```bash
TF_LOG=DEBUG TF_LOG_PATH=tf.log terraform apply
terraform console
terraform graph | dot -Tpng > graph.png
```

---

## 4) Variables, Locals & Outputs

`variables.tf`

```hcl
variable "aws_region" { type = string, default = "ap-south-1" }
variable "tags" {
  type = map(string)
  default = { project="demo", owner="Atul" }
}
```

`terraform.tfvars`

```hcl
aws_region = "ap-south-1"
tags = { project="demo", owner="Atul" }
```

CLI overrides:

```bash
terraform apply -var 'aws_region=us-east-1'
terraform apply -var-file=prod.tfvars
```

Locals:

```hcl
locals {
  prefix = "${var.project}-${terraform.workspace}"
}
```

Outputs:

```hcl
output "website_url" {
  value = "http://${aws_s3_bucket.site.bucket_regional_domain_name}"
}
```

---

## 5) Resources, Data Sources & Meta-Arguments

Resource:

```hcl
resource "aws_s3_bucket" "site" {
  bucket = "${local.prefix}-site"
  tags   = var.tags
}
```

Data source:

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*"]
  }
}
```

Meta-args:

```hcl
count, for_each, depends_on, lifecycle
```

---

## 6) Modules

Example reusable VPC module:

```hcl
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}
```

Pull/update:

```bash
terraform get -update=true
```

---

## 7) Workspaces

```bash
terraform workspace new dev
terraform workspace select dev
terraform workspace list
terraform.workspace   # reference in code
```

---

## 8) State Management

Inspect/move/import:

```bash
terraform show
terraform state list
terraform state show aws_s3_bucket.site
terraform state mv aws_iam_role.old module.iam.aws_iam_role.new
terraform state rm aws_s3_bucket.legacy
terraform state pull > terraform.tfstate
terraform state push terraform.tfstate
terraform state replace-provider hashicorp/aws registry.opentofu.org/aws
```

Import existing:

```bash
terraform import aws_s3_bucket.imported my-existing-bucket
```

---

## 9) Terraform Cloud / Remote Runs

```bash
terraform login
terraform logout
```

Configure with `cloud {}` or `backend "remote" {}`.

---

## 10) Example: AWS Static Website

`main.tf`:

```hcl
resource "aws_s3_bucket" "site" {
  bucket = "tf-site-${terraform.workspace}"
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
terraform init -backend-config=backend.hcl
terraform workspace new dev
terraform plan -out=plan.tfplan
terraform apply plan.tfplan
terraform output url
```

---

## 11) Extended Commands

Taint & unlock:

```bash
terraform taint aws_instance.my_ec2
terraform untaint aws_instance.my_ec2
terraform force-unlock LOCK_ID
```

Console:

```bash
echo 'join(",",["foo","bar"])' | terraform console
```

Graph:

```bash
terraform graph | dot -Tpng > graph.png
```

---

## 12) Best Practices

* Pin provider versions & commit `.terraform.lock.hcl`.
* Never commit `*.tfstate` or secrets.
* Use `for_each` instead of `count` for stable keys.
* Minimize use of `-target`.
* Use `terraform plan -refresh-only` for drift detection.
* Limit concurrency with `-parallelism=N`.

---

## 13) One-Page Command Reference

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
terraform taint|untaint <ADDR>
terraform force-unlock <LOCK_ID>
```

---

## Author & Links

* 📘 [Terraform Cheatsheet](https://atulkamble.github.io/Terraform-Commands-Cheatsheet/)
* 👨‍💻 [GitHub: atulkamble](https://github.com/atulkamble)
* 💼 [LinkedIn: atuljkamble](https://linkedin.com/in/atuljkamble)
* 📝 [Medium: atuljkamble](https://atuljkamble.medium.com)

