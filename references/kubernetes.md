# Kubernetes Reference — EKS (AWS) & AKS (Azure)

## Table of Contents
1. [EKS Cluster Module](#eks)
2. [AKS Cluster Module](#aks)
3. [Node Group Patterns](#node-groups)
4. [IRSA / Workload Identity](#workload-identity)
5. [Add-ons & Tooling](#addons)
6. [Common Patterns & Decisions](#patterns)

---

## 1. EKS Cluster Module {#eks}

### versions.tf
```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws        = { source = "hashicorp/aws",   version = "~> 5.0" }
    kubernetes = { source = "hashicorp/kubernetes", version = "~> 2.23" }
    helm       = { source = "hashicorp/helm",  version = "~> 2.11" }
  }
}
```

### main.tf — EKS Control Plane
```hcl
# IAM Role for EKS Control Plane
resource "aws_iam_role" "eks_cluster" {
  name = "${var.project}-${var.environment}-eks-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
    }]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster.name
}

resource "aws_eks_cluster" "main" {
  name     = "${var.project}-${var.environment}-eks"
  version  = var.kubernetes_version
  role_arn = aws_iam_role.eks_cluster.arn

  vpc_config {
    subnet_ids              = var.private_subnet_ids  # Control plane in private subnets
    endpoint_private_access = true   # Private API endpoint
    endpoint_public_access  = var.enable_public_endpoint  # false in production
    public_access_cidrs     = var.enable_public_endpoint ? var.allowed_public_cidrs : []
    security_group_ids      = [aws_security_group.eks_cluster.id]
  }

  # Encrypt secrets at rest with KMS
  encryption_config {
    provider { key_arn = var.kms_key_arn }
    resources = ["secrets"]
  }

  enabled_cluster_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]

  tags = local.common_tags

  depends_on = [aws_iam_role_policy_attachment.eks_cluster_policy]
}

# EKS Cluster Security Group
resource "aws_security_group" "eks_cluster" {
  name        = "${var.project}-${var.environment}-eks-cluster-sg"
  description = "EKS cluster control plane security group"
  vpc_id      = var.vpc_id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, { Name = "${var.project}-${var.environment}-eks-cluster-sg" })
}

# OIDC Provider for IRSA
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}

resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = aws_eks_cluster.main.identity[0].oidc[0].issuer
  tags            = local.common_tags
}
```

### Node Groups
```hcl
# System node group — for kube-system workloads (CoreDNS, etc.)
resource "aws_eks_node_group" "system" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project}-${var.environment}-system"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = var.private_subnet_ids
  instance_types  = ["m5.large"]  # Fixed size for system workloads

  scaling_config {
    desired_size = 2
    min_size     = 2
    max_size     = 4
  }

  # Taint system nodes so only system pods schedule here
  taint {
    key    = "CriticalAddonsOnly"
    value  = "true"
    effect = "NO_SCHEDULE"
  }

  update_config { max_unavailable = 1 }

  launch_template {
    id      = aws_launch_template.eks_nodes.id
    version = aws_launch_template.eks_nodes.latest_version
  }

  tags = local.common_tags
  depends_on = [aws_iam_role_policy_attachment.eks_node_policies]
}

# Application node group — for workloads (with Cluster Autoscaler)
resource "aws_eks_node_group" "app" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${var.project}-${var.environment}-app"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = var.private_subnet_ids
  instance_types  = var.app_node_instance_types  # ["m5.xlarge", "m5a.xlarge"]
  capacity_type   = "ON_DEMAND"  # Use SPOT for non-critical with fallback

  scaling_config {
    desired_size = var.app_node_desired
    min_size     = var.app_node_min
    max_size     = var.app_node_max
  }

  # Cluster Autoscaler tags
  labels = {
    role = "application"
  }

  update_config { max_unavailable_percentage = 25 }

  launch_template {
    id      = aws_launch_template.eks_nodes.id
    version = aws_launch_template.eks_nodes.latest_version
  }

  tags = merge(local.common_tags, {
    # Required for Cluster Autoscaler discovery
    "k8s.io/cluster-autoscaler/${aws_eks_cluster.main.name}" = "owned"
    "k8s.io/cluster-autoscaler/enabled"                      = "true"
  })
}

# Shared launch template (EBS encryption, IMDSv2)
resource "aws_launch_template" "eks_nodes" {
  name_prefix = "${var.project}-${var.environment}-eks-node-"

  # Enforce IMDSv2 (security baseline)
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2 only
    http_put_response_hop_limit = 1
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 50
      volume_type           = "gp3"
      encrypted             = true
      kms_key_id            = var.kms_key_arn
      delete_on_termination = true
    }
  }

  tag_specifications {
    resource_type = "instance"
    tags          = merge(local.common_tags, { Name = "${var.project}-${var.environment}-eks-node" })
  }

  tags = local.common_tags
}

# Node IAM Role
resource "aws_iam_role" "eks_nodes" {
  name = "${var.project}-${var.environment}-eks-node-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
  tags = local.common_tags
}

locals {
  node_policies = [
    "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy",
    "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy",
    "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly",
    "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore",  # For SSM access instead of SSH
  ]
}

resource "aws_iam_role_policy_attachment" "eks_node_policies" {
  for_each   = toset(local.node_policies)
  policy_arn = each.value
  role       = aws_iam_role.eks_nodes.name
}
```

---

## 2. AKS Cluster Module {#aks}

### main.tf
```hcl
resource "azurerm_resource_group" "aks" {
  name     = "rg-${var.project}-${var.environment}-aks"
  location = var.location
  tags     = local.common_tags
}

# User-assigned identity for AKS (preferred over service principal)
resource "azurerm_user_assigned_identity" "aks" {
  name                = "id-${var.project}-${var.environment}-aks"
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  tags                = local.common_tags
}

# Grant AKS identity Network Contributor on the subnet
resource "azurerm_role_assignment" "aks_network" {
  principal_id         = azurerm_user_assigned_identity.aks.principal_id
  role_definition_name = "Network Contributor"
  scope                = var.aks_subnet_id
}

resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-${var.project}-${var.environment}"
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  dns_prefix          = "${var.project}-${var.environment}"
  kubernetes_version  = var.kubernetes_version

  # Private cluster (no public API endpoint in production)
  private_cluster_enabled             = var.private_cluster
  private_dns_zone_id                 = var.private_cluster ? "System" : null

  # System node pool (required, can't be removed)
  default_node_pool {
    name                = "system"
    node_count          = 2
    vm_size             = "Standard_D2s_v5"
    vnet_subnet_id      = var.aks_subnet_id
    zones               = ["1", "2", "3"]
    os_disk_type        = "Ephemeral"   # Better performance, lower cost
    os_disk_size_gb     = 128
    only_critical_addons_enabled = true  # System pool for system pods only

    upgrade_settings { max_surge = "33%" }

    node_labels = { "role" = "system" }
  }

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.aks.id]
  }

  # Azure CNI for production (not kubenet)
  network_profile {
    network_plugin      = "azure"
    network_policy      = "azure"        # or "calico"
    load_balancer_sku   = "standard"
    outbound_type       = "userAssignedNATGateway"
    service_cidr        = "172.16.0.0/16"
    dns_service_ip      = "172.16.0.10"
  }

  # Azure AD RBAC (use Azure AD groups for access)
  azure_active_directory_role_based_access_control {
    managed            = true
    azure_rbac_enabled = true
    admin_group_object_ids = var.aks_admin_group_ids
  }

  # Disable local accounts (security baseline)
  local_account_disabled = true

  # Auto-upgrade channel
  automatic_upgrade_channel = "patch"
  node_os_upgrade_channel   = "NodeImage"

  # Key Vault secrets provider (for workload identity)
  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    secret_rotation_interval = "2m"
  }

  oms_agent {
    log_analytics_workspace_id      = var.log_analytics_workspace_id
    msi_auth_for_monitoring_enabled = true
  }

  tags = local.common_tags
}

# Application node pool
resource "azurerm_kubernetes_cluster_node_pool" "app" {
  name                  = "app"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = var.app_node_sku
  node_count            = null  # Use autoscaling
  vnet_subnet_id        = var.aks_subnet_id
  zones                 = ["1", "2", "3"]
  os_disk_type          = "Ephemeral"
  os_disk_size_gb       = 128

  auto_scaling_enabled = true
  min_count            = var.app_min_count
  max_count            = var.app_max_count

  node_labels = { "role" = "application" }
  node_taints = []

  upgrade_settings { max_surge = "33%" }

  tags = local.common_tags
}
```

---

## 3. IRSA / Workload Identity {#workload-identity}

### AWS — IRSA (IAM Roles for Service Accounts)
```hcl
# Generic IRSA role factory
resource "aws_iam_role" "irsa" {
  for_each = var.irsa_roles  # map of name -> {namespace, sa_name, policy_arns}

  name = "${var.project}-${var.environment}-${each.key}-irsa"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.eks.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" =
            "system:serviceaccount:${each.value.namespace}:${each.value.sa_name}"
          "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud" =
            "sts.amazonaws.com"
        }
      }
    }]
  })

  tags = merge(local.common_tags, { Name = "${each.key}-irsa" })
}
```

### Azure — Workload Identity (Federated Credentials)
```hcl
resource "azurerm_user_assigned_identity" "workload" {
  for_each            = var.workload_identities  # map of name -> {namespace, sa_name}
  name                = "id-${var.project}-${var.environment}-${each.key}"
  location            = var.location
  resource_group_name = var.resource_group_name
  tags                = local.common_tags
}

resource "azurerm_federated_identity_credential" "workload" {
  for_each            = var.workload_identities
  name                = "fedcred-${each.key}"
  resource_group_name = var.resource_group_name
  parent_id           = azurerm_user_assigned_identity.workload[each.key].id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = azurerm_kubernetes_cluster.main.oidc_issuer_url
  subject             = "system:serviceaccount:${each.value.namespace}:${each.value.sa_name}"
}
```

---

## 4. Key Add-ons {#addons}

### AWS EKS Add-ons (via Terraform)
```hcl
locals {
  eks_addons = {
    "coredns"            = { version = "v1.11.1-eksbuild.4" }
    "kube-proxy"         = { version = "v1.29.0-eksbuild.1" }
    "vpc-cni"            = { version = "v1.16.0-eksbuild.1" }
    "aws-ebs-csi-driver" = { version = "v1.26.0-eksbuild.1" }
  }
}

resource "aws_eks_addon" "main" {
  for_each      = local.eks_addons
  cluster_name  = aws_eks_cluster.main.name
  addon_name    = each.key
  addon_version = each.value.version

  resolve_conflicts_on_update = "OVERWRITE"
  service_account_role_arn    = try(aws_iam_role.irsa[each.key].arn, null)

  tags = local.common_tags
}
```

---

## 5. Common Patterns & Decisions {#patterns}

| Decision | AWS EKS | Azure AKS |
|----------|---------|-----------|
| Networking | VPC CNI (native pods) | Azure CNI (native pods) |
| Auth | AWS IAM + RBAC | Azure AD + Azure RBAC |
| Workload identity | IRSA | Workload Identity (OIDC) |
| Node autoscaling | Cluster Autoscaler or Karpenter | Cluster Autoscaler (built-in) |
| Private registry | ECR | ACR (attach at cluster creation) |
| Secrets | Secrets Store CSI + ASM/SSM | Secrets Store CSI + Key Vault |
| Ingress | AWS Load Balancer Controller | Application Gateway Ingress / nginx |
| Storage | EBS CSI (RWO), EFS CSI (RWX) | Azure Disk (RWO), Azure Files (RWX) |

## Gotchas

- **EKS**: Always use managed node groups over self-managed — they handle AMI updates and drain safely
- **EKS**: `vpc-cni` addon requires IRSA with `AmazonEKS_CNI_Policy` — without it, pod networking breaks after apply
- **AKS**: `local_account_disabled = true` means you **must** have Azure AD groups set up before applying — otherwise you'll be locked out
- **AKS**: Private clusters require private DNS zone or custom DNS — plan this before enabling `private_cluster_enabled`
- **Both**: Kubernetes version upgrades — always upgrade control plane first, then node groups, one minor version at a time
- **Both**: Use `prevent_destroy = true` lifecycle on the cluster resource in production
