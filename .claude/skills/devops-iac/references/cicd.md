# CI/CD Reference — Terraform Pipelines

## Table of Contents
1. [GitHub Actions — AWS](#github-actions-aws)
2. [GitHub Actions — Azure](#github-actions-azure)
3. [Azure DevOps Pipelines — Terraform](#azure-devops)
4. [Atlantis — Pull Request Automation](#atlantis)
5. [Drift Detection](#drift)
6. [State Management](#state)

---

## 1. GitHub Actions — AWS {#github-actions-aws}

### OIDC-based auth (no long-lived keys)
```hcl
# Terraform: create OIDC provider for GitHub Actions
resource "aws_iam_openid_connect_provider" "github_actions" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
  tags            = local.common_tags
}

resource "aws_iam_role" "github_actions" {
  name = "${var.project}-${var.environment}-github-actions"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github_actions.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringLike = {
          "token.actions.githubusercontent.com:sub" =
            "repo:${var.github_org}/${var.github_repo}:environment:${var.environment}"
        }
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })

  tags = local.common_tags
}
```

### GitHub Actions Workflow — Terraform Plan + Apply
```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
    paths: ['infrastructure/**']
  pull_request:
    branches: [main]
    paths: ['infrastructure/**']

permissions:
  id-token: write   # Required for OIDC
  contents: read
  pull-requests: write  # For PR comments

env:
  TF_VERSION: "1.7.0"
  AWS_REGION: "us-east-1"
  WORKING_DIR: "infrastructure/environments/prod"

jobs:
  terraform:
    name: Terraform ${{ github.event_name == 'push' && 'Apply' || 'Plan' }}
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}

    defaults:
      run:
        working-directory: ${{ env.WORKING_DIR }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="bucket=${{ secrets.TF_STATE_BUCKET }}" \
            -backend-config="key=${{ env.WORKING_DIR }}/terraform.tfstate" \
            -backend-config="region=${{ env.AWS_REGION }}" \
            -backend-config="dynamodb_table=${{ secrets.TF_LOCK_TABLE }}"

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        continue-on-error: true

      - name: Comment PR with plan
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📋
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            *Workflow: \`${{ github.workflow }}\` | Actor: \`${{ github.actor }}\`*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

---

## 2. GitHub Actions — Azure {#github-actions-azure}

### Federated credential setup (no client secrets)
```hcl
# Terraform: Service Principal + Federated Credential
resource "azuread_application" "github_actions" {
  display_name = "${var.project}-${var.environment}-github-actions"
}

resource "azuread_service_principal" "github_actions" {
  client_id = azuread_application.github_actions.client_id
}

resource "azuread_application_federated_identity_credential" "github_actions" {
  application_id = azuread_application.github_actions.id
  display_name   = "github-actions-${var.environment}"
  audiences      = ["api://AzureADTokenExchange"]
  issuer         = "https://token.actions.githubusercontent.com"
  subject        = "repo:${var.github_org}/${var.github_repo}:environment:${var.environment}"
}

resource "azurerm_role_assignment" "github_actions" {
  principal_id         = azuread_service_principal.github_actions.object_id
  role_definition_name = "Contributor"
  scope                = "/subscriptions/${var.subscription_id}"
}
```

### GitHub Actions Workflow — Azure Terraform
```yaml
# .github/workflows/terraform-azure.yml
name: Terraform Azure

on:
  push:
    branches: [main]
    paths: ['infra-azure/**']
  pull_request:
    branches: [main]
    paths: ['infra-azure/**']

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  terraform:
    runs-on: ubuntu-latest
    environment: production

    defaults:
      run:
        working-directory: infra-azure/environments/prod

    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init
        run: terraform init
        env:
          ARM_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
          ARM_TENANT_ID:       ${{ secrets.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_USE_OIDC:        "true"

      - name: Terraform Plan
        run: terraform plan -no-color -out=tfplan
        env:
          ARM_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
          ARM_TENANT_ID:       ${{ secrets.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_USE_OIDC:        "true"

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
        env:
          ARM_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
          ARM_TENANT_ID:       ${{ secrets.AZURE_TENANT_ID }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_USE_OIDC:        "true"
```

---

## 3. Azure DevOps Pipelines {#azure-devops}

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main]
  paths:
    include: ['infrastructure/**']

pr:
  branches:
    include: [main]
  paths:
    include: ['infrastructure/**']

variables:
  - group: terraform-prod-vars      # Variable group with ARM_CLIENT_ID etc.
  - name: TF_VERSION
    value: "1.7.0"
  - name: WORKING_DIR
    value: "infrastructure/environments/prod"

pool:
  vmImage: ubuntu-latest

stages:
  - stage: Validate
    jobs:
      - job: TerraformPlan
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: $(TF_VERSION)

          - task: TerraformTaskV4@4
            displayName: Terraform Init
            inputs:
              provider: azurerm
              command: init
              workingDirectory: $(WORKING_DIR)
              backendServiceArm: $(AZURE_SERVICE_CONNECTION)  # ARM service connection
              backendAzureRmResourceGroupName: rg-terraform-state
              backendAzureRmStorageAccountName: $(TF_STATE_STORAGE)
              backendAzureRmContainerName: tfstate
              backendAzureRmKey: prod.terraform.tfstate

          - task: TerraformTaskV4@4
            displayName: Terraform Plan
            inputs:
              provider: azurerm
              command: plan
              workingDirectory: $(WORKING_DIR)
              environmentServiceNameAzureRM: $(AZURE_SERVICE_CONNECTION)
              commandOptions: "-no-color -out=tfplan"

          - publish: $(WORKING_DIR)/tfplan
            artifact: tfplan

  - stage: Apply
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    dependsOn: Validate

    jobs:
      - deployment: TerraformApply
        environment: production         # Requires approval gate in ADO
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: tfplan

                - task: TerraformTaskV4@4
                  displayName: Terraform Apply
                  inputs:
                    provider: azurerm
                    command: apply
                    workingDirectory: $(WORKING_DIR)
                    environmentServiceNameAzureRM: $(AZURE_SERVICE_CONNECTION)
                    commandOptions: "$(Pipeline.Workspace)/tfplan/tfplan"
```

---

## 4. Atlantis {#atlantis}

### atlantis.yaml (repo config)
```yaml
version: 3
automerge: false
delete_source_branch_on_merge: false

projects:
  - name: aws-prod-networking
    dir: infrastructure/aws/networking
    workspace: prod
    terraform_version: v1.7.0
    autoplan:
      when_modified: ["*.tf", "../../../modules/**/*.tf"]
      enabled: true
    apply_requirements:
      - approved
      - mergeable

  - name: aws-prod-eks
    dir: infrastructure/aws/eks
    workspace: prod
    terraform_version: v1.7.0
    autoplan:
      when_modified: ["*.tf", "../../../modules/**/*.tf"]
      enabled: true
    apply_requirements:
      - approved
      - mergeable

  - name: azure-prod-networking
    dir: infrastructure/azure/networking
    workspace: prod
    terraform_version: v1.7.0
    autoplan:
      when_modified: ["*.tf"]
      enabled: true
    apply_requirements:
      - approved
      - mergeable
```

---

## 5. Drift Detection {#drift}

### Scheduled drift check workflow (GitHub Actions)
```yaml
# .github/workflows/drift-detection.yml
name: Drift Detection

on:
  schedule:
    - cron: '0 6 * * 1-5'  # Weekdays at 6 AM UTC
  workflow_dispatch:

permissions:
  id-token: write
  contents: read
  issues: write

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [prod, staging]
        module: [networking, eks, security]

    steps:
      - uses: actions/checkout@v4

      - name: Configure credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_{0}', matrix.environment)] }}
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure/aws/${{ matrix.module }}

      - name: Terraform Plan (drift check)
        id: plan
        run: |
          terraform plan -detailed-exitcode -no-color 2>&1 | tee plan_output.txt
          echo "exit_code=${PIPESTATUS[0]}" >> $GITHUB_OUTPUT
        working-directory: infrastructure/aws/${{ matrix.module }}
        continue-on-error: true

      - name: Open issue on drift detected
        if: steps.plan.outputs.exit_code == '2'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('infrastructure/aws/${{ matrix.module }}/plan_output.txt', 'utf8');

            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `🚨 Drift detected: ${{ matrix.environment }}/${{ matrix.module }}`,
              body: `## Drift Detected\n\n**Environment**: ${{ matrix.environment }}\n**Module**: ${{ matrix.module }}\n\n\`\`\`\n${plan.slice(0, 60000)}\n\`\`\``,
              labels: ['drift', 'infrastructure', '${{ matrix.environment }}']
            });
```

---

## 6. State Management {#state}

### AWS S3 Backend (with DynamoDB locking)
```hcl
# Bootstrap: create state bucket (apply once with local backend, then migrate)
resource "aws_s3_bucket" "terraform_state" {
  bucket = "${var.project}-terraform-state-${var.account_id}"
  tags   = local.common_tags

  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.main.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "${var.project}-terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.main.arn
  }

  tags = local.common_tags
}

# backend.tf (in each environment)
terraform {
  backend "s3" {
    bucket         = "myproject-terraform-state-123456789"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "alias/myproject-prod"
    dynamodb_table = "myproject-terraform-locks"
  }
}
```

### Azure Storage Backend
```hcl
resource "azurerm_storage_account" "terraform_state" {
  name                     = "st${var.project}tfstate"  # Globally unique, no dashes, max 24 chars
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "GRS"  # Geo-redundant

  # Security baseline
  https_traffic_only_enabled      = true
  min_tls_version                 = "TLS1_2"
  public_network_access_enabled   = false  # Private access only
  allow_nested_items_to_be_public = false

  blob_properties {
    versioning_enabled = true
    delete_retention_policy { days = 90 }
  }

  identity { type = "SystemAssigned" }

  tags = local.common_tags

  lifecycle { prevent_destroy = true }
}

resource "azurerm_storage_container" "tfstate" {
  name                  = "tfstate"
  storage_account_name  = azurerm_storage_account.terraform_state.name
  container_access_type = "private"
}

# backend.tf (in each environment)
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stmyprojecttfstate"
    container_name       = "tfstate"
    key                  = "prod/networking/terraform.tfstate"
    use_oidc             = true  # Use OIDC auth in CI/CD
  }
}
```

## Gotchas

- **State locking**: Always use DynamoDB (AWS) or blob lease (Azure, built-in) — without it, concurrent applies corrupt state
- **State in PRs**: Never run `terraform apply` from PRs — only plan on PR, apply on merge to main
- **OIDC auth**: Much safer than long-lived keys — set up OIDC providers before your first pipeline run
- **Atlantis**: Requires a webhook secret and HTTPS endpoint — use a dedicated VM or ECS task, not serverless (too slow)
- **Drift detection**: Run on a schedule, not just on pushes — infra can drift from manual console changes
- **State migration**: When changing backends, use `terraform init -migrate-state` — never delete state files
