# terraform-azurerm-vnet

## Description

This Terraform module deploys a number of shared services that are common to a hub in a hub and spoke environment. Work in progress.

> I may rework this into a number of simple reference terraform files as it has dubious reuse. It will definitely change!!!

* Azure Key Vault
  * Optional public SSH keys
* Log Analytics Workspace
* Recovery Vault
  * Default backup policy

## Usage

### Hub Example

```terraform
provider "azurerm" {
  version = "~> 2.11.0"
  features {}
}

variable "hub" {
  type    = string
  default = "hub"
}

resource "azurerm_resource_group" "hub" {
  name     = var.hub
  location = "West Europe"

  tags     = {
    environment = "dev"
    costcenter  = "it"
  }
}

module "network" {
  source                 = "github.com/terraform-azurerm-modules/terraform-azure-vnet"
  resource_group_name    = azurerm_resource_group.hub.name
  location               = azurerm_resource_group.hub.location
  tags                   = azurerm_resource_group.hub.tags

  vnet_name              = var.hub
  address_space          = [ "10.0.0.0/24" ]
  dns_servers            = [ "10.0.0.68", "10.0.0.69" ]

  subnets                = {
    AzureFirewallSubnet  = "10.0.0.0/26"
    SharedServices       = "10.0.0.64/26"
    AzureBastionSubnet   = "10.0.0.192/27"
    GatewaySubnet        = "10.0.0.224/27"
  }
}

module "shared_services" {
  source                 = "github.com/terraform-azurerm-modules/terraform-azure-shared-services"
  module_depends_on      = azurerm_resource_group.hub
  resource_group_name    = azurerm_resource_group.hub.name
  location               = azurerm_resource_group.hub.location
  tags                   = azurerm_resource_group.hub.tags

  key_vault_name = "${var.hub}-key-vault"
  ssh_public_keys = [
    { username = "ubuntu", ssh_public_key_file = "~/.ssh/id_rsa.pub" },
    { username = "richeney", ssh_public_key_file = "~/.ssh/id_rsa.pub" }
  ]

  workspace_name              = "${var.hub}-workspace"
  recovery_vault_name         = "${var.hub}-recovery-vault"
  diagnostics_storage_account = "${var.hub}bootdiags"
}

output "vnet" {
  value       = module.network.vnet
  description = "The module's vnet object."
}

output "subnets" {
  value       = module.network.subnets
  description = "The module's subnets object."
}

output "hub_workspace" {
  value = module.shared_services.workspace
}

output "hub_key_vault" {
  value = module.shared_services.key_vault
}

output "hub_diags" {
  value = module.shared_services.diags
}

output "ssh_users" {
  value = module.shared_services.ssh_users
}
```

## Authors

Originally created by [Richard Cheney](http://github.com/richeney)

## License

[MIT](LICENSE)
