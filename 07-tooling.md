# Tooling (Azure CLI & Terraform)

## Azure CLI
A cross-platform command-line tool (Windows, macOS, Linux) to automate Azure resources.

**Common Commands**:
- `az login` / `az login --use-device-code`: Authenticate.
- `az account list --output table`: View subscriptions.
- `az group create --name <name> --location centralindia`: Create a Resource Group.
- `az vm list --output table`: List VMs.
- `az network nsg rule list -g <rg> --nsg-name <nsg>`: View open ports.

## Terraform
Infrastructure as Code (IaC) tool for multi-cloud provisioning.
- **State Management**: Keeps track of resources in a `terraform.tfstate` file.
- **Modularity**: Allows reuse of code for multiple environments.

**Common Commands**:
- `terraform init`: Downloads the Azure provider and initializes the backend.
- `terraform plan`: Dry run showing what will be created/changed/deleted.
- `terraform apply`: Executes the changes.
- `terraform destroy`: Permanently deletes the resources.
- `terraform fmt`: Fixes formatting.
- `terraform validate`: Checks syntax errors.
