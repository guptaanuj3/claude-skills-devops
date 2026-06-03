# Terraform Core Reference — Modules, Workspaces, Providers

## Table of Contents
1. [Repository Structure](#repo-structure)
2. [Provider Configuration](#providers)
3. [Module Patterns](#modules)
4. [Workspaces & Environment Strategy](#workspaces)
5. [Variables & tfvars Strategy](#variables)
6. [Terraform Versions & Constraints](#versions)

---

## 1. Repository Structure {#repo-structure}

### Monorepo Layout (recommended for most teams)
```
infrastructure/
├── modules/                    # Reusable modules (no backend, no provider)
│   ├── aws/
│   │   ├── vpc/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── versions.tf
│   │   │   └── README.md
│   │   ├── eks/
│   │   └── rds/
│   └── azure/
│       ├── vnet/
│       ├── aks/
│       └── keyvault/
│
├── environments/
│   ├── aws/
│   │   ├── dev/
│   │   │   ├── networking/
│   │   │   │   ├── main.tf       # Calls modules, wires outputs
│   │   │   │   ├── backend.tf    # Remote state config
│   │   │   │   ├── versions.tf   # Provider + terraform versions
│   │   │   │   └── terraform.tfvars
│   │   │   ├── eks/
│   │   │   └── security/
│   │   ├── staging/
│   │   └── prod/
│   └── azure/
│       ├── dev/
│       ├── staging/
│       └── prod/
│
├── scripts/
│   ├── bootstrap/              # One-time state bucket creation
│   └── drift-check.sh
│
└── .github/
    └── workflows/
        ├── terraform-aws.yml
        └── terraform-azure.yml
```

---

## 2. Provider Configuration {#providers}

### AWS Provider (multi-region, multi-account)
```hcl
# versions.tf
terraform {
  required_version = ">= 1.5, < 2.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Minor updates allowed, no major
    }
  }
}

# providers.tf
provider "aws" {
  region = var.region

  # Assume deploy role (never use root or admin credentials)
  assume_role {
    role_arn     = "arn:aws:iam::${var.account_id}:role/${var.project}-${var.environment}-terraform"
    session_name = "terraform-${var.environment}"
  }

  default_tags {
    tags = {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "terraform"
      Repository  = var.repo_name
    }
  }
}

# Secondary region alias (for DR, Route53, etc.)
provider "aws" {
  alias  = "secondary"
  region = var.secondary_region

  assume_role {
    role_arn     = "arn:aws:iam::${var.account_id}:role/${var.project}-${var.environment}-terraform"
    session_name = "terraform-${var.environment}-secondary"
  }
}
```

### Azure Provider
```hcl
# versions.tf
terraform {
  required_version = ">= 1.5, < 2.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.90"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.47"
    }
  }
}

# providers.tf
provider "azurerm" {
  features {
    resource_group { prevent_deletion_if_contains_resources = true }
    key_vault {
      purge_soft_delete_on_destroy               = false  # Keep soft-deleted resources
      recover_soft_deleted_key_vaults            = true
    }
    virtual_machine { delete_os_disk_on_deletion = true }
  }

  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
  use_oidc        = true  # OIDC auth in CI/CD; falls back to az CLI locally
}

provider "azuread" {
  tenant_id = var.tenant_id
  use_oidc  = true
}
```

---

## 3. Module Patterns {#modules}

### Calling a module from an environment
```hcl
# environments/aws/prod/networking/main.tf

module "vpc" {
  source = "../../../../modules/aws/vpc"  # Local path (prefer over registry for internal modules)

  # Pass all required variables explicitly
  project              = var.project
  environment          = var.environment
  owner                = var.owner
  region               = var.region
  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]
  data_subnet_cidrs    = ["10.0.20.0/24", "10.0.21.0/24", "10.0.22.0/24"]
  kms_key_arn          = module.security.kms_key_arn
}

module "eks" {
  source = "../../../../modules/aws/eks"

  project            = var.project
  environment        = var.environment
  owner              = var.owner
  vpc_id             = module.vpc.vpc_id               # Wire VPC outputs to EKS
  private_subnet_ids = module.vpc.private_subnet_ids
  kms_key_arn        = module.security.kms_key_arn
  kubernetes_version = "1.29"
  app_node_min       = 2
  app_node_max       = 10
  app_node_desired   = 3
}
```

### Remote state data source (cross-module, cross-environment)
```hcl
# Read outputs from another environment's state
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "${var.project}-terraform-state-${var.account_id}"
    key    = "${var.environment}/networking/terraform.tfstate"
    region = var.region
  }
}

# Use the remote VPC ID
module "eks" {
  source  = "../../../../modules/aws/eks"
  vpc_id  = data.terraform_remote_state.networking.outputs.vpc_id
  # ...
}
```

---

## 4. Workspaces & Environment Strategy {#workspaces}

### Recommended: Directories over workspaces
Use **separate directories** per environment (as shown in repo structure above), not Terraform workspaces.

Reason: Workspaces share the same code path — a mistake in workspace selection can destroy prod.
Separate directories mean separate state files, separate backend configs, separate CI/CD jobs.

### When to use workspaces
Workspaces are appropriate for **ephemeral environments** (feature branches, PR previews):

```hcl
# In CI/CD for PR previews
locals {
  env_suffix = terraform.workspace == "default" ? "main" : terraform.workspace
}

resource "aws_eks_cluster" "main" {
  name = "${var.project}-${local.env_suffix}-eks"
  # ...
}
```

```bash
# Create and select workspace for PR
terraform workspace new pr-123
terraform workspace select pr-123
terraform apply -var="environment=pr-123"

# Destroy on PR close
terraform destroy -auto-approve
terraform workspace select default
terraform workspace delete pr-123
```

---

## 5. Variables & tfvars Strategy {#variables}

### variables.tf — Always include description and type
```hcl
variable "project" {
  type        = string
  description = "Project name used as prefix for all resources"
  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,20}$", var.project))
    error_message = "Project must be lowercase alphanumeric with hyphens, 3-21 chars."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "region" {
  type        = string
  description = "AWS region or Azure location"
}

variable "owner" {
  type        = string
  description = "Team or person responsible for this infrastructure"
}
```

### tfvars layout
```
environments/aws/prod/networking/
├── terraform.tfvars          # Non-sensitive defaults (committed)
└── secrets.auto.tfvars       # NEVER committed — injected by CI/CD from vault
```

```hcl
# terraform.tfvars (committed)
project     = "myapp"
environment = "prod"
region      = "us-east-1"
owner       = "platform-team"
vpc_cidr    = "10.0.0.0/16"
```

```hcl
# secrets.auto.tfvars (NOT committed, injected by CI/CD)
db_password    = "injected-from-secrets-manager"
slack_webhook  = "injected-from-secrets-manager"
```

---

## 6. Versions & Constraints {#versions}

### Pinning strategy
```hcl
terraform {
  required_version = ">= 1.5, < 2.0"  # Floor + ceiling
  required_providers {
    aws    = { source = "hashicorp/aws",    version = "~> 5.0"  }  # Allow patch
    azurerm= { source = "hashicorp/azurerm",version = "~> 3.90" }
    helm   = { source = "hashicorp/helm",   version = "~> 2.11" }
    kubernetes = { source = "hashicorp/kubernetes", version = "~> 2.23" }
  }
}
```

### .terraform-version (for tfenv/tofuenv)
```
1.7.5
```

### Upgrade process
1. Update version constraint in `versions.tf`
2. Run `terraform init -upgrade` in a feature branch
3. Run `terraform plan` — review for provider behavior changes
4. Apply to dev → staging → prod sequentially

## Gotchas

- **Module source**: Use relative paths (`../../modules/vpc`) for internal modules — registry sources require versioning infrastructure
- **Remote state**: Never use `depends_on` between root modules — pass values explicitly via outputs/variables
- **`terraform.tfvars`**: Automatically loaded — don't name other var files `terraform.tfvars` unless you want them auto-applied
- **Workspaces + state**: Each workspace gets its own state file in the backend — don't use workspace-based state separation for truly isolated environments
- **Provider version upgrades**: Major version bumps (e.g., azurerm 3.x → 4.x) often have breaking changes — read the changelog before upgrading
- **`-target`**: Avoid `terraform apply -target` in production — it creates partial state that's hard to reconcile
