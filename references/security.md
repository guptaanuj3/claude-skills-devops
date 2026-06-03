# Security Reference — IAM (AWS) & RBAC / Key Vault (Azure)

## Table of Contents
1. [AWS IAM — Roles, Policies, SCPs](#aws-iam)
2. [Azure RBAC & Managed Identities](#azure-rbac)
3. [KMS (AWS) / Key Vault (Azure)](#encryption)
4. [Secrets Management](#secrets)
5. [Security Baselines & Guardrails](#baselines)

---

## 1. AWS IAM {#aws-iam}

### Principle: Roles, not users. Always condition-scoped.

```hcl
# Cross-account assumable role (e.g., CI/CD from tooling account)
resource "aws_iam_role" "cicd_deploy" {
  name = "${var.project}-${var.environment}-cicd-deploy"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::${var.tooling_account_id}:root" }
      Action    = "sts:AssumeRole"
      Condition = {
        StringEquals = { "sts:ExternalId" = var.external_id }
        Bool         = { "aws:MultiFactorAuthPresent" = "true" }  # Require MFA for human access
      }
    }]
  })

  max_session_duration = 3600  # 1 hour max

  tags = local.common_tags
}

# Scoped deployment policy (no wildcards on actions or resources)
resource "aws_iam_policy" "cicd_deploy" {
  name        = "${var.project}-${var.environment}-cicd-deploy-policy"
  description = "Allows CI/CD to deploy to EKS and push to ECR"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ECRPush"
        Effect = "Allow"
        Action = [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:BatchCheckLayerAvailability",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload"
        ]
        Resource = "arn:aws:ecr:${var.region}:${var.account_id}:repository/${var.project}-*"
      },
      {
        Sid      = "ECRLogin"
        Effect   = "Allow"
        Action   = "ecr:GetAuthorizationToken"
        Resource = "*"
      },
      {
        Sid    = "EKSDescribe"
        Effect = "Allow"
        Action = ["eks:DescribeCluster"]
        Resource = "arn:aws:eks:${var.region}:${var.account_id}:cluster/${var.project}-${var.environment}-*"
      }
    ]
  })
}

# Permission Boundary — cap max permissions for any role in the account
resource "aws_iam_policy" "permission_boundary" {
  name = "${var.project}-permission-boundary"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid      = "AllowScopedServices"
        Effect   = "Allow"
        Action   = ["ec2:*", "eks:*", "ecr:*", "s3:*", "logs:*", "kms:Decrypt", "kms:GenerateDataKey"]
        Resource = "*"
      },
      {
        # Deny IAM privilege escalation
        Sid    = "DenyIAMEscalation"
        Effect = "Deny"
        Action = [
          "iam:CreateUser", "iam:AttachUserPolicy", "iam:PutUserPolicy",
          "iam:CreatePolicy", "iam:AttachRolePolicy"
        ]
        Resource = "*"
      }
    ]
  })
}
```

### SCPs (Service Control Policies) — AWS Organizations
```hcl
resource "aws_organizations_policy" "deny_root_actions" {
  name        = "deny-root-actions"
  description = "Prevent root account actions in all member accounts"
  type        = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyRootActions"
        Effect    = "Deny"
        Action    = "*"
        Resource  = "*"
        Condition = { StringLike = { "aws:PrincipalArn" = "arn:aws:iam::*:root" } }
      },
      {
        Sid      = "DenyLeaveOrganization"
        Effect   = "Deny"
        Action   = "organizations:LeaveOrganization"
        Resource = "*"
      },
      {
        # Restrict to approved regions only
        Sid    = "DenyUnapprovedRegions"
        Effect = "Deny"
        Action = "*"
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" = var.approved_regions  # e.g. ["us-east-1", "eu-west-1"]
          }
        }
        NotAction = [
          # Global services must be excluded from region restriction
          "iam:*", "organizations:*", "support:*", "sts:*", "cloudfront:*", "route53:*"
        ]
      }
    ]
  })
}
```

---

## 2. Azure RBAC & Managed Identities {#azure-rbac}

### Managed Identity — Preferred over Service Principals
```hcl
# User-assigned identity for an application
resource "azurerm_user_assigned_identity" "app" {
  name                = "id-${var.project}-${var.environment}-${var.app_name}"
  location            = var.location
  resource_group_name = var.resource_group_name
  tags                = local.common_tags
}

# Scope role assignments to specific resources (not subscription-wide)
resource "azurerm_role_assignment" "app_kv_secrets" {
  principal_id         = azurerm_user_assigned_identity.app.principal_id
  role_definition_name = "Key Vault Secrets User"  # Read-only secrets
  scope                = var.key_vault_id
}

resource "azurerm_role_assignment" "app_storage_reader" {
  principal_id         = azurerm_user_assigned_identity.app.principal_id
  role_definition_name = "Storage Blob Data Reader"
  scope                = "${var.storage_account_id}/blobServices/default/containers/${var.container_name}"
  # Scope to specific container, not entire storage account
}
```

### Custom RBAC Role (least privilege)
```hcl
resource "azurerm_role_definition" "aks_deployer" {
  name        = "${var.project}-${var.environment}-aks-deployer"
  scope       = "/subscriptions/${var.subscription_id}"
  description = "Allows CI/CD to push images to ACR and deploy to AKS"

  permissions {
    actions = [
      "Microsoft.ContainerRegistry/registries/push/write",
      "Microsoft.ContainerRegistry/registries/pull/read",
      "Microsoft.ContainerService/managedClusters/listClusterUserCredential/action",
      "Microsoft.ContainerService/managedClusters/read"
    ]
    not_actions = []
  }

  assignable_scopes = [
    "/subscriptions/${var.subscription_id}/resourceGroups/${var.resource_group_name}"
  ]
}
```

### Azure AD PIM (Privileged Identity Management) via Terraform
```hcl
# Eligible assignment (JIT access) — requires Azure AD Premium P2
resource "azurerm_pim_eligible_role_assignment" "owner_jit" {
  scope              = "/subscriptions/${var.subscription_id}"
  role_definition_id = data.azurerm_role_definition.owner.role_definition_id
  principal_id       = var.break_glass_group_object_id

  justification = "Break-glass emergency access"

  schedule {
    expiration {
      duration_type  = "Hour"
      duration       = 8
    }
  }
}
```

---

## 3. Encryption — KMS (AWS) & Key Vault (Azure) {#encryption}

### AWS KMS — Multi-region, key rotation
```hcl
resource "aws_kms_key" "main" {
  description              = "${var.project}-${var.environment} CMK"
  key_usage                = "ENCRYPT_DECRYPT"
  customer_master_key_spec = "SYMMETRIC_DEFAULT"
  enable_key_rotation      = true   # Annual auto-rotation
  deletion_window_in_days  = 30     # Max safety window
  multi_region             = var.multi_region_key

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EnableRootAccess"
        Effect = "Allow"
        Principal = { AWS = "arn:aws:iam::${var.account_id}:root" }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "AllowServices"
        Effect = "Allow"
        Principal = { Service = ["logs.amazonaws.com", "s3.amazonaws.com"] }
        Action   = ["kms:GenerateDataKey*", "kms:Decrypt"]
        Resource = "*"
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_kms_alias" "main" {
  name          = "alias/${var.project}-${var.environment}"
  target_key_id = aws_kms_key.main.key_id
}
```

### Azure Key Vault — with private endpoint
```hcl
resource "azurerm_key_vault" "main" {
  name                       = "kv-${var.project}-${var.environment}"  # Max 24 chars
  location                   = var.location
  resource_group_name        = var.resource_group_name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "premium"  # HSM-backed keys

  # Disable public access (security baseline)
  public_network_access_enabled = false
  network_acls {
    default_action = "Deny"
    bypass         = "AzureServices"
  }

  enable_rbac_authorization       = true   # Use RBAC, not access policies
  purge_protection_enabled        = true   # Prevent accidental deletion
  soft_delete_retention_days      = 90

  tags = local.common_tags
}

# Private Endpoint for Key Vault
resource "azurerm_private_endpoint" "key_vault" {
  name                = "pe-${var.project}-${var.environment}-kv"
  location            = var.location
  resource_group_name = var.resource_group_name
  subnet_id           = var.data_subnet_id

  private_service_connection {
    name                           = "kv-connection"
    private_connection_resource_id = azurerm_key_vault.main.id
    subresource_names              = ["vault"]
    is_manual_connection           = false
  }

  tags = local.common_tags
}

# Key Vault Key for disk/DB encryption
resource "azurerm_key_vault_key" "cmk" {
  name         = "${var.project}-${var.environment}-cmk"
  key_vault_id = azurerm_key_vault.main.id
  key_type     = "RSA-HSM"
  key_size     = 4096

  key_opts = ["decrypt", "encrypt", "sign", "unwrapKey", "verify", "wrapKey"]

  rotation_policy {
    automatic {
      time_before_expiry = "P30D"  # Auto-rotate 30 days before expiry
    }
    expire_after         = "P1Y"   # 1-year key lifetime
    notify_before_expiry = "P29D"
  }
}
```

---

## 4. Secrets Management {#secrets}

### AWS — SSM Parameter Store / Secrets Manager
```hcl
# Use Secrets Manager for credentials, SSM for config
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "/${var.project}/${var.environment}/db/password"
  kms_key_id              = aws_kms_key.main.id
  recovery_window_in_days = 30  # Prevent accidental deletion

  tags = local.common_tags
}

# Reference in application via IRSA — never put the value in Terraform state
# terraform.tfvars should never contain actual secrets
```

### Azure — Reference secrets in Terraform without storing them
```hcl
# Read a secret that was set outside Terraform
data "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  key_vault_id = azurerm_key_vault.main.id
}

# Pass to resource (value stays in Key Vault, only reference in state)
resource "azurerm_postgresql_flexible_server" "main" {
  administrator_password = data.azurerm_key_vault_secret.db_password.value
  # ... other config
}
```

---

## 5. Security Baselines {#baselines}

### AWS Mandatory Controls
```hcl
# S3 Block Public Access (account-level)
resource "aws_s3_account_public_access_block" "main" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# EBS Default Encryption
resource "aws_ebs_encryption_by_default" "main" {
  enabled = true
}

# GuardDuty
resource "aws_guardduty_detector" "main" {
  enable = true
  datasources {
    s3_logs            { enable = true }
    kubernetes         { audit_logs { enable = true } }
    malware_protection { scan_ec2_instance_with_findings { ebs_volumes { enable = true } } }
  }
  tags = local.common_tags
}

# CloudTrail (account-wide, encrypted, immutable)
resource "aws_cloudtrail" "main" {
  name                          = "${var.project}-${var.environment}-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  kms_key_id                    = aws_kms_key.main.arn
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true  # Detect tampering
  tags                          = local.common_tags
}
```

### Azure Mandatory Controls
```hcl
# Defender for Cloud — enable all plans
resource "azurerm_security_center_subscription_pricing" "plans" {
  for_each      = toset(["VirtualMachines", "SqlServers", "AppServices", "KeyVaults", "Arm", "Containers", "Dns"])
  tier          = "Standard"
  resource_type = each.value
}

# Activity Log — 1-year retention
resource "azurerm_monitor_diagnostic_setting" "subscription" {
  name               = "diag-subscription-activity"
  target_resource_id = "/subscriptions/${var.subscription_id}"
  log_analytics_workspace_id = var.log_analytics_workspace_id

  enabled_log { category = "Administrative" }
  enabled_log { category = "Security" }
  enabled_log { category = "Policy" }
  enabled_log { category = "Alert" }
}

# Azure Policy — enforce tagging
resource "azurerm_policy_assignment" "require_tags" {
  name                 = "require-tags"
  scope                = "/subscriptions/${var.subscription_id}"
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025"
  display_name         = "Require tags on resource groups"
}
```

## Gotchas

- **AWS**: Never attach `AdministratorAccess` to a role used by automation — scope to minimum needed actions
- **AWS**: KMS key deletion cannot be cancelled after the waiting period — always use `deletion_window_in_days = 30`
- **Azure**: `enable_rbac_authorization = true` on Key Vault means access policies are ignored — pick one model and stick to it
- **Azure**: Purge protection on Key Vault cannot be disabled once enabled — plan this before creating
- **Both**: Terraform state contains sensitive values (passwords, connection strings) — always use encrypted remote backends with access controls
