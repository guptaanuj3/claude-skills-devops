# Networking Reference — AWS & Azure

## Table of Contents
1. [AWS VPC — Full Stack Module](#aws-vpc)
2. [Azure VNet — Full Stack Module](#azure-vnet)
3. [VPC/VNet Peering](#peering)
4. [Transit Gateway (AWS) / Virtual WAN (Azure)](#hub-spoke)
5. [DNS Configuration](#dns)
6. [Common Patterns & Decisions](#patterns)

---

## 1. AWS VPC — Full Stack Module {#aws-vpc}

### Recommended CIDR Layout (3-tier, multi-AZ)
```
VPC: 10.0.0.0/16
  Public  subnets: 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24   (one per AZ)
  Private subnets: 10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24 (workloads)
  Data    subnets: 10.0.20.0/24, 10.0.21.0/24, 10.0.22.0/24 (RDS, ElastiCache)
```

### main.tf
```hcl
data "aws_availability_zones" "available" { state = "available" }

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true   # Required for EKS and private hosted zones
  tags                 = merge(local.common_tags, { Name = "${var.project}-${var.environment}-vpc" })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = merge(local.common_tags, { Name = "${var.project}-${var.environment}-igw" })
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = false  # Never auto-assign public IPs

  tags = merge(local.common_tags, {
    Name                     = "${var.project}-${var.environment}-public-${count.index + 1}"
    "kubernetes.io/role/elb" = "1"  # Required if using EKS with public ALB
  })
}

# Private Subnets (workloads)
resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = merge(local.common_tags, {
    Name                              = "${var.project}-${var.environment}-private-${count.index + 1}"
    "kubernetes.io/role/internal-elb" = "1"  # Required for EKS internal ALB
  })
}

# Data Subnets (isolated — no route to NAT)
resource "aws_subnet" "data" {
  count             = length(var.data_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.data_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.project}-${var.environment}-data-${count.index + 1}"
  })
}

# NAT Gateways (one per AZ for HA)
resource "aws_eip" "nat" {
  count  = length(var.public_subnet_cidrs)
  domain = "vpc"
  tags   = merge(local.common_tags, { Name = "${var.project}-${var.environment}-eip-${count.index + 1}" })
}

resource "aws_nat_gateway" "main" {
  count         = length(var.public_subnet_cidrs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  tags          = merge(local.common_tags, { Name = "${var.project}-${var.environment}-nat-${count.index + 1}" })
  depends_on    = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = merge(local.common_tags, { Name = "${var.project}-${var.environment}-rt-public" })
}

resource "aws_route_table" "private" {
  count  = length(var.private_subnet_cidrs)
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id  # Each AZ uses its own NAT
  }
  tags = merge(local.common_tags, { Name = "${var.project}-${var.environment}-rt-private-${count.index + 1}" })
}

resource "aws_route_table" "data" {
  vpc_id = aws_vpc.main.id
  # No default route — data subnet is isolated
  tags   = merge(local.common_tags, { Name = "${var.project}-${var.environment}-rt-data" })
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(var.private_subnet_cidrs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

resource "aws_route_table_association" "data" {
  count          = length(var.data_subnet_cidrs)
  subnet_id      = aws_subnet.data[count.index].id
  route_table_id = aws_route_table.data.id
}

# VPC Flow Logs (security baseline requirement)
resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name              = "/aws/vpc/${var.project}-${var.environment}"
  retention_in_days = 90
  kms_key_id        = var.kms_key_arn  # Encrypt flow logs at rest
  tags              = local.common_tags
}

resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_logs.arn
  tags            = local.common_tags
}
```

### variables.tf
```hcl
variable "project"              { type = string }
variable "environment"          { type = string }
variable "owner"                { type = string }
variable "vpc_cidr"             { type = string; default = "10.0.0.0/16" }
variable "public_subnet_cidrs"  { type = list(string) }
variable "private_subnet_cidrs" { type = list(string) }
variable "data_subnet_cidrs"    { type = list(string) }
variable "kms_key_arn"          { type = string; description = "KMS key for encrypting flow logs" }
```

### outputs.tf
```hcl
output "vpc_id"              { value = aws_vpc.main.id }
output "public_subnet_ids"   { value = aws_subnet.public[*].id }
output "private_subnet_ids"  { value = aws_subnet.private[*].id }
output "data_subnet_ids"     { value = aws_subnet.data[*].id }
output "nat_gateway_ids"     { value = aws_nat_gateway.main[*].id }
```

---

## 2. Azure VNet — Full Stack Module {#azure-vnet}

### Recommended CIDR Layout
```
VNet: 10.1.0.0/16
  snet-public:  10.1.0.0/24   (Application Gateway, Front Door origin)
  snet-app:     10.1.10.0/23  (AKS node pools, App Service)
  snet-data:    10.1.20.0/24  (Azure SQL, Redis, private endpoints)
  snet-mgmt:    10.1.30.0/28  (Bastion, jump hosts)
```

### main.tf
```hcl
resource "azurerm_resource_group" "network" {
  name     = "rg-${var.project}-${var.environment}-network"
  location = var.location
  tags     = local.common_tags
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.project}-${var.environment}"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  address_space       = [var.vnet_cidr]
  tags                = local.common_tags
}

resource "azurerm_subnet" "app" {
  name                 = "snet-app"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.app_subnet_cidr]

  # Required for AKS
  service_endpoints = ["Microsoft.ContainerRegistry", "Microsoft.KeyVault"]
}

resource "azurerm_subnet" "data" {
  name                 = "snet-data"
  resource_group_name  = azurerm_resource_group.network.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [var.data_subnet_cidr]

  # Private endpoints for PaaS services (SQL, Redis, Storage)
  private_endpoint_network_policies = "Disabled"
}

# Network Security Group for App subnet
resource "azurerm_network_security_group" "app" {
  name                = "nsg-${var.project}-${var.environment}-app"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name

  security_rule {
    name                       = "deny-internet-inbound"
    priority                   = 4000
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "Internet"
    destination_address_prefix = "*"
  }

  tags = local.common_tags
}

resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id                 = azurerm_subnet.app.id
  network_security_group_id = azurerm_network_security_group.app.id
}

# NAT Gateway for outbound (private subnets)
resource "azurerm_public_ip" "nat" {
  name                = "pip-${var.project}-${var.environment}-nat"
  location            = azurerm_resource_group.network.location
  resource_group_name = azurerm_resource_group.network.name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]
  tags                = local.common_tags
}

resource "azurerm_nat_gateway" "main" {
  name                    = "natgw-${var.project}-${var.environment}"
  location                = azurerm_resource_group.network.location
  resource_group_name     = azurerm_resource_group.network.name
  sku_name                = "Standard"
  idle_timeout_in_minutes = 10
  zones                   = ["1"]
  tags                    = local.common_tags
}

resource "azurerm_nat_gateway_public_ip_association" "main" {
  nat_gateway_id       = azurerm_nat_gateway.main.id
  public_ip_address_id = azurerm_public_ip.nat.id
}

resource "azurerm_subnet_nat_gateway_association" "app" {
  subnet_id      = azurerm_subnet.app.id
  nat_gateway_id = azurerm_nat_gateway.main.id
}

# Network Watcher Flow Logs (security baseline)
resource "azurerm_network_watcher_flow_log" "app" {
  network_watcher_name = "NetworkWatcher_${var.location}"
  resource_group_name  = "NetworkWatcherRG"
  name                 = "flowlog-${var.project}-${var.environment}-app"

  network_security_group_id = azurerm_network_security_group.app.id
  storage_account_id        = var.flow_log_storage_account_id
  enabled                   = true

  retention_policy {
    enabled = true
    days    = 90
  }

  traffic_analytics {
    enabled               = true
    workspace_id          = var.log_analytics_workspace_id
    workspace_region      = var.location
    workspace_resource_id = var.log_analytics_workspace_resource_id
  }

  tags = local.common_tags
}
```

---

## 3. VPC/VNet Peering {#peering}

### AWS — Cross-Account VPC Peering
```hcl
# In the requester account
resource "aws_vpc_peering_connection" "main" {
  vpc_id        = var.local_vpc_id
  peer_vpc_id   = var.peer_vpc_id
  peer_owner_id = var.peer_account_id
  peer_region   = var.peer_region
  auto_accept   = false  # Must be accepted in the peer account
  tags          = merge(local.common_tags, { Name = "pcx-${var.project}-to-${var.peer_name}" })
}

# In the accepter account (separate provider alias)
resource "aws_vpc_peering_connection_accepter" "main" {
  provider                  = aws.peer
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id
  auto_accept               = true
}

# Add routes in both VPCs (requester side)
resource "aws_route" "to_peer" {
  count                     = length(var.private_route_table_ids)
  route_table_id            = var.private_route_table_ids[count.index]
  destination_cidr_block    = var.peer_vpc_cidr
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id
}
```

### Azure — VNet Peering
```hcl
resource "azurerm_virtual_network_peering" "local_to_remote" {
  name                      = "peer-${var.local_vnet_name}-to-${var.remote_vnet_name}"
  resource_group_name       = var.local_rg
  virtual_network_name      = var.local_vnet_name
  remote_virtual_network_id = var.remote_vnet_id

  allow_virtual_network_access = true
  allow_forwarded_traffic      = true   # Required for hub-spoke
  allow_gateway_transit        = false  # Set true on hub side
  use_remote_gateways          = false  # Set true on spoke side
}

resource "azurerm_virtual_network_peering" "remote_to_local" {
  name                      = "peer-${var.remote_vnet_name}-to-${var.local_vnet_name}"
  resource_group_name       = var.remote_rg
  virtual_network_name      = var.remote_vnet_name
  remote_virtual_network_id = var.local_vnet_id

  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
}
```

---

## 4. Hub-Spoke Topology {#hub-spoke}

### AWS — Transit Gateway
```hcl
resource "aws_ec2_transit_gateway" "main" {
  description                     = "${var.project}-${var.environment} TGW"
  default_route_table_association = "disable"  # Manage routes explicitly
  default_route_table_propagation = "disable"
  auto_accept_shared_attachments  = "disable"

  tags = merge(local.common_tags, { Name = "${var.project}-${var.environment}-tgw" })
}

resource "aws_ec2_transit_gateway_vpc_attachment" "spoke" {
  for_each           = var.spoke_vpcs  # map of name -> {vpc_id, subnet_ids}
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = each.value.vpc_id
  subnet_ids         = each.value.subnet_ids
  tags               = merge(local.common_tags, { Name = "tgw-attach-${each.key}" })
}
```

---

## 5. Common Patterns & Decisions {#patterns}

| Scenario | Recommendation |
|----------|---------------|
| Single region, <5 VPCs | VPC Peering (simpler, cheaper) |
| Multi-region or >5 VPCs | Transit Gateway / Virtual WAN |
| Shared services (DNS, scanning) | Hub-spoke with centralized egress |
| On-prem connectivity | AWS Direct Connect / Azure ExpressRoute + VPN backup |
| AKS/EKS networking | Azure CNI (AKS) or VPC CNI (EKS) — never kubenet in prod |
| Private PaaS access (Azure) | Private Endpoints for all PaaS (SQL, Storage, ACR) |

## Gotchas

- **AWS**: NAT Gateway charges are per-AZ — always put NAT in same AZ as workload subnets to avoid cross-AZ data transfer costs
- **AWS**: EKS requires specific subnet tags (`kubernetes.io/role/internal-elb`) for load balancer discovery — include them at VPC creation time
- **Azure**: NSG rules are evaluated by priority (lower = first); always add a final `Deny All` rule at priority 4096
- **Azure**: VNet peering is non-transitive — spoke VNets cannot reach each other through hub without a firewall or route forwarding
- **Both**: Plan your CIDR ranges before you deploy — re-IPing is painful. Use at least /16 for prod VNets
