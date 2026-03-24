# Infrastructure as Code - Terraform Templates

## 1. Project Structure

```
infrastructure-terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── aks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── redis/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── monitoring/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── dev/
    │   └── terraform.tfvars
    ├── staging/
    │   └── terraform.tfvars
    └── production/
        └── terraform.tfvars
```

## 2. Main Configuration

```hcl
# main.tf
terraform {
  required_version = ">= 1.6.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80.0"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.45.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5.0"
    }
  }
  
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "financial-app.tfstate"
  }
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy = false
      recover_soft_deleted_key_vaults = true
    }
    
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
  }
}

# Resource Groups
resource "azurerm_resource_group" "primary" {
  name     = "${var.project_name}-${var.environment}-rg"
  location = var.primary_region
  
  tags = merge(
    var.common_tags,
    {
      Environment = var.environment
      Region      = "primary"
    }
  )
}

resource "azurerm_resource_group" "secondary" {
  name     = "${var.project_name}-${var.environment}-dr-rg"
  location = var.secondary_region
  
  tags = merge(
    var.common_tags,
    {
      Environment = var.environment
      Region      = "secondary"
    }
  )
}

# Networking Module
module "networking_primary" {
  source = "./modules/networking"
  
  resource_group_name = azurerm_resource_group.primary.name
  location            = azurerm_resource_group.primary.location
  vnet_name           = "${var.project_name}-vnet-primary"
  address_space       = ["10.0.0.0/16"]
  
  subnets = {
    aks = {
      address_prefixes = ["10.0.0.0/20"]
      service_endpoints = ["Microsoft.Sql", "Microsoft.Storage", "Microsoft.KeyVault"]
    }
    appgw = {
      address_prefixes = ["10.0.16.0/24"]
    }
    apim = {
      address_prefixes = ["10.0.17.0/24"]
    }
    data = {
      address_prefixes = ["10.0.18.0/24"]
      service_endpoints = ["Microsoft.Sql", "Microsoft.Storage"]
    }
    bastion = {
      address_prefixes = ["10.0.19.0/27"]
    }
    private_endpoint = {
      address_prefixes = ["10.0.20.0/24"]
    }
  }
  
  tags = var.common_tags
}

module "networking_secondary" {
  source = "./modules/networking"
  
  resource_group_name = azurerm_resource_group.secondary.name
  location            = azurerm_resource_group.secondary.location
  vnet_name           = "${var.project_name}-vnet-secondary"
  address_space       = ["10.1.0.0/16"]
  
  subnets = {
    aks = {
      address_prefixes = ["10.1.0.0/20"]
      service_endpoints = ["Microsoft.Sql", "Microsoft.Storage", "Microsoft.KeyVault"]
    }
    appgw = {
      address_prefixes = ["10.1.16.0/24"]
    }
    apim = {
      address_prefixes = ["10.1.17.0/24"]
    }
    data = {
      address_prefixes = ["10.1.18.0/24"]
      service_endpoints = ["Microsoft.Sql", "Microsoft.Storage"]
    }
    bastion = {
      address_prefixes = ["10.1.19.0/27"]
    }
    private_endpoint = {
      address_prefixes = ["10.1.20.0/24"]
    }
  }
  
  tags = var.common_tags
}

# VNet Peering
resource "azurerm_virtual_network_peering" "primary_to_secondary" {
  name                      = "primary-to-secondary"
  resource_group_name       = azurerm_resource_group.primary.name
  virtual_network_name      = module.networking_primary.vnet_name
  remote_virtual_network_id = module.networking_secondary.vnet_id
  
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
}

resource "azurerm_virtual_network_peering" "secondary_to_primary" {
  name                      = "secondary-to-primary"
  resource_group_name       = azurerm_resource_group.secondary.name
  virtual_network_name      = module.networking_secondary.vnet_name
  remote_virtual_network_id = module.networking_primary.vnet_id
  
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
}

# AKS Module
module "aks_primary" {
  source = "./modules/aks"
  
  resource_group_name = azurerm_resource_group.primary.name
  location            = azurerm_resource_group.primary.location
  cluster_name        = "${var.project_name}-aks-primary"
  dns_prefix          = "${var.project_name}-primary"
  
  kubernetes_version = var.kubernetes_version
  
  default_node_pool = {
    name                = "system"
    node_count          = 3
    vm_size             = "Standard_D4s_v3"
    availability_zones  = ["1", "2", "3"]
    enable_auto_scaling = true
    min_count           = 3
    max_count           = 6
  }
  
  user_node_pools = [
    {
      name                = "userpool"
      node_count          = 6
      vm_size             = "Standard_D8s_v3"
      availability_zones  = ["1", "2", "3"]
      enable_auto_scaling = true
      min_count           = 6
      max_count           = 20
      node_labels = {
        workload = "application"
      }
    }
  ]
  
  network_profile = {
    network_plugin     = "azure"
    network_policy     = "azure"
    service_cidr       = "10.2.0.0/16"
    dns_service_ip     = "10.2.0.10"
    docker_bridge_cidr = "172.17.0.1/16"
  }
  
  vnet_subnet_id = module.networking_primary.subnet_ids["aks"]
  
  enable_azure_policy             = true
  enable_azure_active_directory   = true
  enable_role_based_access_control = true
  
  tags = var.common_tags
}

# Database Module
module "database_primary" {
  source = "./modules/database"
  
  resource_group_name = azurerm_resource_group.primary.name
  location            = azurerm_resource_group.primary.location
  server_name         = "${var.project_name}-db-primary"
  
  administrator_login    = var.db_admin_username
  administrator_password = var.db_admin_password
  
  sku_name   = "GP_Standard_D16s_v3"
  storage_mb = 2097152  # 2TB
  
  backup_retention_days        = 35
  geo_redundant_backup_enabled = true
  
  high_availability = {
    mode                      = "ZoneRedundant"
    standby_availability_zone = "2"
  }
  
  postgresql_version = "15"
  
  firewall_rules = []  # Use private endpoint only
  
  private_endpoint = {
    subnet_id = module.networking_primary.subnet_ids["private_endpoint"]
  }
  
  tags = var.common_tags
}

module "database_secondary" {
  source = "./modules/database"
  
  resource_group_name = azurerm_resource_group.secondary.name
  location            = azurerm_resource_group.secondary.location
  server_name         = "${var.project_name}-db-secondary"
  
  create_mode = "Replica"
  source_server_id = module.database_primary.server_id
  
  sku_name   = "GP_Standard_D16s_v3"
  storage_mb = 2097152
  
  postgresql_version = "15"
  
  private_endpoint = {
    subnet_id = module.networking_secondary.subnet_ids["private_endpoint"]
  }
  
  tags = var.common_tags
}

# Redis Module
module "redis_primary" {
  source = "./modules/redis"
  
  resource_group_name = azurerm_resource_group.primary.name
  location            = azurerm_resource_group.primary.location
  redis_name          = "${var.project_name}-redis-primary"
  
  capacity            = 3
  family              = "P"
  sku_name            = "Premium"
  
  enable_non_ssl_port = false
  minimum_tls_version = "1.2"
  
  redis_configuration = {
    maxmemory_policy = "allkeys-lru"
    rdb_backup_enabled = true
    rdb_backup_frequency = 60
    rdb_backup_max_snapshot_count = 1
  }
  
  zones = ["1", "2", "3"]
  
  private_endpoint = {
    subnet_id = module.networking_primary.subnet_ids["private_endpoint"]
  }
  
  tags = var.common_tags
}

# Monitoring Module
module "monitoring" {
  source = "./modules/monitoring"
  
  resource_group_name = azurerm_resource_group.primary.name
  location            = azurerm_resource_group.primary.location
  
  log_analytics_workspace_name = "${var.project_name}-logs"
  application_insights_name    = "${var.project_name}-insights"
  
  retention_in_days = 90
  
  tags = var.common_tags
}
```

## 3. Networking Module

```hcl
# modules/networking/main.tf
resource "azurerm_virtual_network" "main" {
  name                = var.vnet_name
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = var.address_space
  
  tags = var.tags
}

resource "azurerm_subnet" "subnets" {
  for_each = var.subnets
  
  name                 = each.key
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = each.value.address_prefixes
  service_endpoints    = lookup(each.value, "service_endpoints", [])
  
  dynamic "delegation" {
    for_each = lookup(each.value, "delegation", null) != null ? [each.value.delegation] : []
    content {
      name = delegation.value.name
      service_delegation {
        name    = delegation.value.service_name
        actions = delegation.value.actions
      }
    }
  }
}

resource "azurerm_network_security_group" "aks" {
  name                = "${var.vnet_name}-aks-nsg"
  location            = var.location
  resource_group_name = var.resource_group_name
  
  security_rule {
    name                       = "AllowAppGateway"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_ranges    = ["80", "443"]
    source_address_prefix      = azurerm_subnet.subnets["appgw"].address_prefixes[0]
    destination_address_prefix = "*"
  }
  
  security_rule {
    name                       = "AllowAzureServices"
    priority                   = 110
    direction                  = "Outbound"
    access                     = "Allow"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "AzureCloud"
  }
  
  tags = var.tags
}

resource "azurerm_subnet_network_security_group_association" "aks" {
  subnet_id                 = azurerm_subnet.subnets["aks"].id
  network_security_group_id = azurerm_network_security_group.aks.id
}

# modules/networking/variables.tf
variable "resource_group_name" {
  description = "Name of the resource group"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
}

variable "vnet_name" {
  description = "Name of the virtual network"
  type        = string
}

variable "address_space" {
  description = "Address space for the virtual network"
  type        = list(string)
}

variable "subnets" {
  description = "Map of subnets to create"
  type = map(object({
    address_prefixes  = list(string)
    service_endpoints = optional(list(string), [])
    delegation = optional(object({
      name         = string
      service_name = string
      actions      = list(string)
    }))
  }))
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}

# modules/networking/outputs.tf
output "vnet_id" {
  description = "ID of the virtual network"
  value       = azurerm_virtual_network.main.id
}

output "vnet_name" {
  description = "Name of the virtual network"
  value       = azurerm_virtual_network.main.name
}

output "subnet_ids" {
  description = "Map of subnet names to IDs"
  value       = { for k, v in azurerm_subnet.subnets : k => v.id }
}
```

## 4. AKS Module

```hcl
# modules/aks/main.tf
resource "azurerm_kubernetes_cluster" "main" {
  name                = var.cluster_name
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = var.dns_prefix
  kubernetes_version  = var.kubernetes_version
  
  default_node_pool {
    name                = var.default_node_pool.name
    node_count          = var.default_node_pool.node_count
    vm_size             = var.default_node_pool.vm_size
    availability_zones  = var.default_node_pool.availability_zones
    enable_auto_scaling = var.default_node_pool.enable_auto_scaling
    min_count           = var.default_node_pool.min_count
    max_count           = var.default_node_pool.max_count
    vnet_subnet_id      = var.vnet_subnet_id
    
    upgrade_settings {
      max_surge = "33%"
    }
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  network_profile {
    network_plugin     = var.network_profile.network_plugin
    network_policy     = var.network_profile.network_policy
    service_cidr       = var.network_profile.service_cidr
    dns_service_ip     = var.network_profile.dns_service_ip
    docker_bridge_cidr = var.network_profile.docker_bridge_cidr
    load_balancer_sku  = "standard"
  }
  
  azure_active_directory_role_based_access_control {
    managed                = true
    azure_rbac_enabled     = true
    admin_group_object_ids = var.admin_group_object_ids
  }
  
  azure_policy_enabled = var.enable_azure_policy
  
  oms_agent {
    log_analytics_workspace_id = var.log_analytics_workspace_id
  }
  
  key_vault_secrets_provider {
    secret_rotation_enabled  = true
    secret_rotation_interval = "2m"
  }
  
  tags = var.tags
}

resource "azurerm_kubernetes_cluster_node_pool" "user_pools" {
  for_each = { for pool in var.user_node_pools : pool.name => pool }
  
  name                  = each.value.name
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = each.value.vm_size
  node_count            = each.value.node_count
  availability_zones    = each.value.availability_zones
  enable_auto_scaling   = each.value.enable_auto_scaling
  min_count             = each.value.min_count
  max_count             = each.value.max_count
  vnet_subnet_id        = var.vnet_subnet_id
  node_labels           = each.value.node_labels
  
  upgrade_settings {
    max_surge = "33%"
  }
  
  tags = var.tags
}

# Container Registry
resource "azurerm_container_registry" "main" {
  name                = replace("${var.cluster_name}acr", "-", "")
  resource_group_name = var.resource_group_name
  location            = var.location
  sku                 = "Premium"
  admin_enabled       = false
  
  georeplications {
    location = var.secondary_location
    tags     = var.tags
  }
  
  network_rule_set {
    default_action = "Deny"
    
    ip_rule {
      action   = "Allow"
      ip_range = var.allowed_ip_ranges
    }
  }
  
  tags = var.tags
}

resource "azurerm_role_assignment" "aks_acr_pull" {
  principal_id                     = azurerm_kubernetes_cluster.main.kubelet_identity[0].object_id
  role_definition_name             = "AcrPull"
  scope                            = azurerm_container_registry.main.id
  skip_service_principal_aad_check = true
}
```

## 5. Database Module

```hcl
# modules/database/main.tf
resource "azurerm_postgresql_flexible_server" "main" {
  name                = var.server_name
  resource_group_name = var.resource_group_name
  location            = var.location
  
  administrator_login    = var.create_mode == "Default" ? var.administrator_login : null
  administrator_password = var.create_mode == "Default" ? var.administrator_password : null
  
  sku_name   = var.sku_name
  storage_mb = var.storage_mb
  version    = var.postgresql_version
  
  backup_retention_days        = var.backup_retention_days
  geo_redundant_backup_enabled = var.geo_redundant_backup_enabled
  
  create_mode      = var.create_mode
  source_server_id = var.source_server_id
  
  dynamic "high_availability" {
    for_each = var.high_availability != null ? [var.high_availability] : []
    content {
      mode                      = high_availability.value.mode
      standby_availability_zone = high_availability.value.standby_availability_zone
    }
  }
  
  tags = var.tags
}

resource "azurerm_postgresql_flexible_server_configuration" "ssl" {
  name      = "require_secure_transport"
  server_id = azurerm_postgresql_flexible_server.main.id
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_configuration" "connection_throttling" {
  name      = "connection_throttle.enable"
  server_id = azurerm_postgresql_flexible_server.main.id
  value     = "on"
}

resource "azurerm_postgresql_flexible_server_database" "databases" {
  for_each = toset(var.databases)
  
  name      = each.value
  server_id = azurerm_postgresql_flexible_server.main.id
  charset   = "UTF8"
  collation = "en_US.utf8"
}

resource "azurerm_private_endpoint" "database" {
  count = var.private_endpoint != null ? 1 : 0
  
  name                = "${var.server_name}-pe"
  location            = var.location
  resource_group_name = var.resource_group_name
  subnet_id           = var.private_endpoint.subnet_id
  
  private_service_connection {
    name                           = "${var.server_name}-psc"
    private_connection_resource_id = azurerm_postgresql_flexible_server.main.id
    is_manual_connection           = false
    subresource_names              = ["postgresqlServer"]
  }
  
  tags = var.tags
}
```

## 6. Variables File

```hcl
# variables.tf
variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "financial-app"
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "primary_region" {
  description = "Primary Azure region"
  type        = string
  default     = "eastus2"
}

variable "secondary_region" {
  description = "Secondary Azure region for DR"
  type        = string
  default     = "westus2"
}

variable "kubernetes_version" {
  description = "Kubernetes version"
  type        = string
  default     = "1.28.3"
}

variable "db_admin_username" {
  description = "Database administrator username"
  type        = string
  sensitive   = true
}

variable "db_admin_password" {
  description = "Database administrator password"
  type        = string
  sensitive   = true
}

variable "common_tags" {
  description = "Common tags to apply to all resources"
  type        = map(string)
  default = {
    Project     = "Financial Services"
    ManagedBy   = "Terraform"
    CostCenter  = "Engineering"
    Compliance  = "PCI-DSS"
  }
}
```

## 7. Production Environment Configuration

```hcl
# environments/production/terraform.tfvars
project_name     = "financial-app"
environment      = "production"
primary_region   = "eastus2"
secondary_region = "westus2"

kubernetes_version = "1.28.3"

common_tags = {
  Project     = "Financial Services"
  Environment = "Production"
  ManagedBy   = "Terraform"
  CostCenter  = "Engineering"
  Compliance  = "PCI-DSS"
  Owner       = "Platform Team"
}
```

## 8. Deployment Commands

```bash
# Initialize Terraform
terraform init

# Validate configuration
terraform validate

# Format code
terraform fmt -recursive

# Plan changes
terraform plan -var-file=environments/production/terraform.tfvars -out=tfplan

# Apply changes
terraform apply tfplan

# Destroy resources (use with caution)
terraform destroy -var-file=environments/production/terraform.tfvars
```

## 9. State Management

```bash
# List resources in state
terraform state list

# Show specific resource
terraform state show azurerm_kubernetes_cluster.main

# Import existing resource
terraform import azurerm_resource_group.primary /subscriptions/{sub-id}/resourceGroups/{rg-name}

# Move resource in state
terraform state mv azurerm_resource_group.old azurerm_resource_group.new

# Remove resource from state
terraform state rm azurerm_resource_group.temp
```

## 10. Best Practices

### Security
- Store sensitive variables in Azure Key Vault
- Use managed identities instead of service principals
- Enable private endpoints for all PaaS services
- Implement network security groups and firewall rules
- Use customer-managed keys for encryption

### State Management
- Use remote backend (Azure Storage)
- Enable state locking
- Implement state versioning
- Regular state backups
- Use workspaces for environments

### Code Organization
- Use modules for reusability
- Implement consistent naming conventions
- Add comprehensive documentation
- Use variables for configuration
- Implement validation rules

### CI/CD Integration
- Automate terraform fmt and validate
- Run security scanning (tfsec, Checkov)
- Implement approval gates for production
- Use separate service principals per environment
- Implement drift detection