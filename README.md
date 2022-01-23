# Module Azure AD Pod Identity

Module for installing AAD Pod Identity inside a Kubernetes Clusters with the proper AzureIdentity Bindings.

## Usage

```bash
module "ssh_key" {
  source = "git::https://github.com/danielscholl-terraform/module-ssh-key?ref=v1.0.0"
}

module "resource_group" {
  source = "git::https://github.com/danielscholl-terraform/module-resource-group?ref=v1.0.0"

  name     = "iac-terraform"
  location = "eastus2"

  resource_tags = {
    iac = "terraform"
  }
}

module "aks" {
  source     = "git::https://github.com/danielscholl-terraform/module-aks?ref=v1.0.0"
  depends_on = [module.resource_group, module.ssh_key]

  name                = format("iac-terraform-cluster-%s", module.resource_group.random)
  resource_group_name = module.resource_group.name
  dns_prefix          = format("iac-terraform-cluster-%s", module.resource_group.random)

  linux_profile = {
    admin_username = "k8sadmin"
    ssh_key        = "${trimspace(module.ssh_key.public_ssh_key)} k8sadmin"
  }

  network_plugin    = "azure" ### <-- AAD Pod Identity is not compatable with Kubenet must use Azure.
  default_node_pool = "default"
  node_pools = {
    default = {
      vm_size                = "Standard_B2s"
      enable_host_encryption = true

      node_count = 2
    }
  }

  resource_tags = {
    iac = "terraform"
  }
}

resource "azurerm_user_assigned_identity" "identity" {
  name                = "pod-identity"
  resource_group_name = module.resource_group.name
  location            = module.resource_group.location

  tags = {
    iac = "terraform"
  }
}

module "aad_pod_identity" {
  source     = "../"
  depends_on = [module.aks]

  providers = { helm = helm.aks }

  aks_node_resource_group = module.aks.node_resource_group
  aks_identity            = module.aks.kubelet_identity.object_id

  identities = {
    ## Add an AzureIdentity and Binding for Pod Identity to the cluster
    pod-identity = {
      namespace   = "default"
      name        = azurerm_user_assigned_identity.identity.name
      client_id   = azurerm_user_assigned_identity.identity.client_id
      resource_id = azurerm_user_assigned_identity.identity.id
    }
  }
}
```

<!--- BEGIN_TF_DOCS --->

<!--- END_TF_DOCS --->
