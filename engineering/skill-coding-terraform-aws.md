---
name: coding-terraform-aws
description: >
  Expert Terraform engineering skill for AWS infrastructure. Use this skill whenever the user
  asks to write, generate, scaffold, review, or refactor any Terraform code targeting AWS.
  Triggers include: creating AWS resources (EC2, RDS, S3, Lambda, SQS, VPC, IAM, ECS, etc.),
  writing terraform modules, setting up tfvars, structuring a Terraform project, enforcing
  tagging strategy, reviewing existing .tf files, or any mention of "infrastructure as code",
  "IaC", "Terraform", ".tf file", "tfvars", or "terraform plan/apply". Always use this skill
  even for small snippets — every piece of Terraform code must follow these standards.
---

# Terraform AWS Skill

You are an expert Terraform engineer. Every `.tf` file you write must conform exactly to the
rules in this skill. No exceptions. Read this entire document before generating any code.

---

## 1. Project Structure (Always Use This Layout)

```
project-root/
├── main.tf          # Module calls ONLY — no direct resource blocks
├── variables.tf     # All root input variable declarations
├── outputs.tf       # All root output blocks
├── providers.tf     # provider {} blocks
├── versions.tf      # terraform { required_providers {} }
├── backend.tf       # terraform { backend {} }
├── locals.tf        # locals {} block — name prefix, common_tags, etc.
│
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
│
└── modules/
    └── <component>/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        ├── locals.tf        # if needed
        └── README.md
```

**Hard rules:**
- Root `main.tf` = module calls only. Zero direct `resource` or `data` blocks at root.
- Every module must have `variables.tf`, `outputs.tf`, and `README.md`.
- Never co-locate provider config with resources.

---

## 2. Component Grouping — One Folder Per Component

When implementing any component (JIRA integration, SNS pipeline, RDS cluster, etc.),
**all associated resources live inside a dedicated module folder**. Never scatter resources
across multiple root-level files.

```
# CORRECT
modules/jira_integration/
├── main.tf       ← SQS, Lambda, IAM, EventBridge all here
├── variables.tf
├── outputs.tf
└── README.md

# WRONG — resources scattered across root
main.tf     ← aws_sqs_queue
iam.tf      ← aws_iam_role
lambda.tf   ← aws_lambda_function
```

---

## 3. No Hardcoded Values — Everything From Variables

```hcl
# WRONG
resource "aws_instance" "app" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.medium"
}

# CORRECT
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type
}
```

Every `variables.tf` entry must have `description`, `type`, and validation where applicable:

```hcl
variable "instance_type" {
  description = "EC2 instance type for the application server"
  type        = string

  validation {
    condition     = contains(["t3.small", "t3.medium", "t3.large"], var.instance_type)
    error_message = "Must be t3.small, t3.medium, or t3.large."
  }
}

variable "db_password" {
  description = "Master password for the RDS instance"
  type        = string
  sensitive   = true
}
```

---

## 4. Environment-Separated tfvars

One `.tfvars` file per environment. Never share a single tfvars across environments.

```hcl
# environments/dev.tfvars
project        = "payments-platform"
environment    = "dev"
org_unit       = "engineering"
business_unit  = "platform"
cost_center    = "CC-1042"
owner          = "platform-team@acme.com"
support_center = "ops-support@acme.com"
instance_type  = "t3.small"
min_capacity   = 1

# environments/prod.tfvars
project        = "payments-platform"
environment    = "prod"
org_unit       = "engineering"
business_unit  = "platform"
cost_center    = "CC-1042"
owner          = "platform-team@acme.com"
support_center = "ops-support@acme.com"
instance_type  = "t3.large"
min_capacity   = 3
```

Invoke with:
```bash
terraform plan  -var-file="environments/dev.tfvars"
terraform apply -var-file="environments/prod.tfvars"
```

---

## 5. Mandatory Tagging Strategy

**Every taggable AWS resource must carry exactly these tags.** No more, no fewer mandatory tags.
Additional resource-specific tags are merged on top with `merge()`.

### Root variables.tf — Declare All Tag Variables

```hcl
variable "project" {
  description = "Project name"
  type        = string
}

variable "org_unit" {
  description = "Organisational unit (e.g. engineering, finance)"
  type        = string
}

variable "business_unit" {
  description = "Business unit responsible for the resource"
  type        = string
}

variable "cost_center" {
  description = "Cost center code for billing allocation"
  type        = string
}

variable "desc" {
  description = "Short human-readable description of the resource group"
  type        = string
}

variable "environment" {
  description = "Deployment environment: dev | staging | prod"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod."
  }
}

variable "owner" {
  description = "Email address of the team or individual who owns this resource"
  type        = string
}

variable "support_center" {
  description = "Support contact or ticket queue for this resource"
  type        = string
}
```

### locals.tf — Define createdby and common_tags

```hcl
locals {
  createdby = "terraform"

  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    project       = var.project
    orgunit       = var.org_unit
    businessunit  = var.business_unit
    costcenter    = var.cost_center
    desc          = var.desc
    createdby     = local.createdby
    environment   = var.environment
    owner         = var.owner
    supportcenter = var.support_center
  }
}
```

### Applying Tags on Resources

**Singleton resource:**
```hcl
resource "aws_sqs_queue" "this" {
  name = "${local.name_prefix}-events"
  # ... other arguments ...
  tags = local.common_tags
}
```

**for_each resource — merge common_tags with per-instance tags:**
```hcl
resource "aws_sqs_queue" "this" {
  for_each = var.sqs_queues

  name                       = "${local.name_prefix}-${each.key}"
  visibility_timeout_seconds = each.value.visibility_timeout_seconds
  message_retention_seconds  = each.value.message_retention_seconds

  tags = merge(local.common_tags, each.value.tags, {
    Name = "${local.name_prefix}-${each.key}"
  })
}
```

**Passing tags into modules:**
```hcl
# Root main.tf
module "jira_integration" {
  source = "./modules/jira_integration"
  # ...
  tags   = local.common_tags
}

# modules/jira_integration/variables.tf
variable "tags" {
  description = "Tags to apply to all resources in this module"
  type        = map(string)
  default     = {}
}
```

---

## 6. Map Objects + for_each — Multi-Instance Provisioning

Never duplicate resource blocks. Use `map(object(...))` variables + `for_each`.

```hcl
# variables.tf
variable "sqs_queues" {
  description = "Map of SQS queues to provision"
  type = map(object({
    delay_seconds              = number
    max_message_size           = number
    message_retention_seconds  = number
    visibility_timeout_seconds = number
    fifo_queue                 = bool
    tags                       = map(string)
  }))
  default = {}
}

# environments/dev.tfvars
sqs_queues = {
  orders = {
    delay_seconds              = 0
    max_message_size           = 262144
    message_retention_seconds  = 86400
    visibility_timeout_seconds = 30
    fifo_queue                 = false
    tags                       = { Purpose = "order-processing" }
  }
  notifications = {
    delay_seconds              = 5
    max_message_size           = 131072
    message_retention_seconds  = 43200
    visibility_timeout_seconds = 60
    fifo_queue                 = true
    tags                       = { Purpose = "user-notifications" }
  }
}

# modules/sqs/main.tf
resource "aws_sqs_queue" "this" {
  for_each = var.sqs_queues

  name                       = "${local.name_prefix}-${each.key}"
  delay_seconds              = each.value.delay_seconds
  max_message_size           = each.value.max_message_size
  message_retention_seconds  = each.value.message_retention_seconds
  visibility_timeout_seconds = each.value.visibility_timeout_seconds
  fifo_queue                 = each.value.fifo_queue

  tags = merge(var.tags, each.value.tags, {
    Name = "${local.name_prefix}-${each.key}"
  })
}
```

**count vs for_each decision:**
| Scenario | Rule |
|---|---|
| Toggle a resource on/off | `count = var.enabled ? 1 : 0` |
| Multiple named resources with distinct config | `for_each = var.resource_map` |
| Iterating a list | `for_each = toset(var.list)` — never `count` on a list |

---

## 7. Module Design Rules

- Root `main.tf` calls modules only.
- Every module exposes a `tags` variable of type `map(string)`.
- No region, account ID, or environment name hardcoded inside modules — passed as variables.
- Local modules use relative paths: `source = "./modules/<name>"`
- External registry modules must be version-pinned:
  ```hcl
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"
  ```

---

## 8. Provider & Version Pinning (versions.tf)

```hcl
terraform {
  required_version = ">= 1.6.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}
```

---

## 9. Remote State Backend (backend.tf)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "<project>/<environment>/<component>/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

State key convention: `<project>/<environment>/<component>/terraform.tfstate`

---

## 10. AWS Resource Best Practices

### IAM
- Least-privilege always. Never `Action = "*"` and `Resource = "*"` together.
- Use `aws_iam_policy_document` data source — never raw JSON strings.
- Attach policies to roles, not users.

```hcl
data "aws_iam_policy_document" "lambda_policy" {
  statement {
    sid    = "AllowSQSConsume"
    effect = "Allow"
    actions = [
      "sqs:ReceiveMessage",
      "sqs:DeleteMessage",
      "sqs:GetQueueAttributes",
    ]
    resources = [aws_sqs_queue.this.arn]
  }
}
```

### S3 (AWS Provider ≥ 4.x — sub-resources only)
```hcl
resource "aws_s3_bucket" "this" {
  bucket        = var.bucket_name
  force_destroy = var.force_destroy
  tags          = var.tags
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Lambda
- Always set explicit `timeout` and `memory_size`.
- Always define `dead_letter_config`.
- Sensitive config via SSM/Secrets Manager — not plaintext environment variables.

### RDS
- `deletion_protection = true` in prod.
- `backup_retention_period` ≥ 7 days for prod.
- `publicly_accessible = false` always.
- Use `vpc_security_group_ids`, not the deprecated `security_groups`.

### Networking
- Never hardcode VPC/subnet IDs — use data sources.
```hcl
data "aws_vpc" "selected" {
  tags = { Name = var.vpc_name }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.selected.id]
  }
  tags = { Tier = "private" }
}
```

---

## 11. Deprecated Attributes — Never Use These

| ❌ Deprecated | ✅ Replacement |
|---|---|
| Inline `acl` in `aws_s3_bucket` | `aws_s3_bucket_acl` resource |
| Inline `versioning` in `aws_s3_bucket` | `aws_s3_bucket_versioning` resource |
| Inline `logging` in `aws_s3_bucket` | `aws_s3_bucket_logging` resource |
| Inline `website` in `aws_s3_bucket` | `aws_s3_bucket_website_configuration` resource |
| `aws_alb` / `aws_alb_listener` | `aws_lb` / `aws_lb_listener` |
| `security_groups` in `aws_db_instance` | `vpc_security_group_ids` |
| `number_cache_clusters` in ElastiCache | `num_cache_clusters` |
| `template_file` data source | `templatefile()` function |
| Raw JSON string in `assume_role_policy` | `aws_iam_policy_document` data source |

---

## 12. Code Readability & Formatting

- Run `terraform fmt` — treat as mandatory.
- 2-space indentation.
- Align `=` signs when 3+ sequential assignments exist.
- Resource names: `snake_case`, purpose-driven, not type-driven.
  - ❌ `aws_lambda_function.lambda`
  - ✅ `aws_lambda_function.jira_ticket_sync`
- Singleton resources within a module: use `this` as the name.
- Argument order within a block:
  1. `count` / `for_each`
  2. Identity (`name`, `bucket`, `role`)
  3. Core functional arguments
  4. Network/security (`vpc_id`, `subnet_ids`, `security_group_ids`)
  5. Feature flags / optional settings
  6. `depends_on`
  7. `lifecycle`
  8. `tags`

---

## 13. Secrets & Security

```hcl
# Pull secrets at runtime — never store in tfvars
data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = var.db_secret_arn
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)
}
```

**.gitignore must include:**
```
*.tfvars
*.tfvars.json
!environments/example.tfvars
.terraform/
*.tfstate
*.tfstate.backup
crash.log
```

---

## 14. Output Standards

```hcl
output "queue_arns" {
  description = "Map of SQS queue ARNs keyed by queue name"
  value       = { for k, q in aws_sqs_queue.this : k => q.arn }
}

output "lambda_arn" {
  description = "ARN of the Lambda function"
  value       = aws_lambda_function.this.arn
  sensitive   = false
}
```

- Every output must have a `description`.
- Mark `sensitive = true` for any output containing credentials, keys, or tokens.
- Expose all cross-component integration points from root outputs.

---

## 15. Pre-Generation Checklist

Before emitting any Terraform code, verify every item:

```
STRUCTURE
[ ] Root main.tf has only module calls
[ ] Component is self-contained in its own module folder
[ ] Module has variables.tf, outputs.tf, README.md

VARIABLES & TFVARS
[ ] Zero hardcoded values — all from variables
[ ] All variables have description + type
[ ] Sensitive variables marked sensitive = true
[ ] Separate .tfvars per environment (dev / staging / prod)

MODULES
[ ] Duplicate resources extracted to a module
[ ] for_each used instead of duplicate resource blocks
[ ] map(object(...)) used for multi-instance provisioning

TAGGING
[ ] All 9 mandatory tag keys present on every taggable resource
[ ] common_tags defined in locals.tf using exact keys:
      project, orgunit, businessunit, costcenter, desc,
      createdby, environment, owner, supportcenter
[ ] createdby = local.createdby (i.e. "terraform") — not a variable
[ ] merge() used to add per-resource/per-instance tags on top

AWS BEST PRACTICES
[ ] IAM uses aws_iam_policy_document, not raw JSON
[ ] S3 uses sub-resources only (no inline blocks)
[ ] All storage services have encryption enabled
[ ] RDS: publicly_accessible = false
[ ] Lambda: timeout + memory_size + dead_letter_config set
[ ] Networking: VPC/subnet IDs from data sources

TERRAFORM BEST PRACTICES
[ ] Provider and Terraform versions pinned in versions.tf
[ ] Remote backend with DynamoDB lock in backend.tf
[ ] locals used for name_prefix and common_tags

DEPRECATED ATTRIBUTES
[ ] No inline S3 blocks (acl, versioning, logging, website)
[ ] aws_lb used instead of aws_alb
[ ] vpc_security_group_ids used instead of security_groups on RDS
[ ] aws_iam_policy_document used — no raw JSON strings

SECURITY
[ ] .gitignore covers *.tfvars
[ ] No secrets in code or .tfvars
[ ] Secrets fetched from Secrets Manager or SSM at runtime
```

---

## 16. Complete Reference — Tagging Variables in tfvars

Every environment `.tfvars` file must supply all tag-related variables:

```hcl
# environments/prod.tfvars  (all tag variables required)
project       = "payments-platform"
environment   = "prod"
org_unit      = "engineering"
business_unit = "platform"
cost_center   = "CC-1042"
desc          = "Payment processing infrastructure for the checkout product"
owner         = "platform-team@acme.com"
support_center = "ops-support@acme.com"
```

The `createdby` tag is **never** supplied via tfvars — it is always the static local value
`"terraform"` set inside `locals.tf`. This ensures it cannot be accidentally overridden.
