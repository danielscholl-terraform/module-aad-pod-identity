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
## Providers

| Name | Version |
|------|---------|
| azurerm | >= 2.90.0 |
| helm | >= 2.4.1 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:-----:|
| additional\_scopes | aad pod identity scopes residing outside of AKS MC\_resource\_group (resource group id or identity id would be a common input) | `map(string)` | `{}` | no |
| additional\_yaml\_config | n/a | `string` | `""` | no |
| aks\_identity | Service principal client\_id or kubelet identity client\_id. See [here](https://github.com/Azure/aad-pod-identity/blob/master/website/content/en/docs/Getting%20started/role-assignment.md). | `string` | n/a | yes |
| aks\_node\_resource\_group | resource group created by AKS | `string` | n/a | yes |
| create\_kubernetes\_namespace | Create the namespace for the identity if it doesn't yet exist | `bool` | `true` | no |
| enable\_kubenet\_plugin | Enable feature when AKS cluster uses Kubenet network plugin, leave default if use AzureCNI | `bool` | `false` | no |
| helm\_chart\_version | Azure AD pod identity helm chart version | `string` | `"3.0.3"` | no |
| identities | Azure identities to be configured | <pre>map(object({<br>    namespace   = string<br>    name        = string<br>    client_id   = string<br>    resource_id = string<br>  }))</pre> | n/a | yes |
| install\_crds | Install CRDs | `bool` | `true` | no |
| kubernetes\_namespace | kubernetes namespace | `string` | `"aad-pod-identity"` | no |

## Outputs

No output.
<!--- END_TF_DOCS --->
