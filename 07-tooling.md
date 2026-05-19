# Tooling (Azure CLI & Terraform)

## The Core Pillars of DevOps
Before diving into the tools, a Senior Cloud Engineer must understand that tooling is only part of the equation. A mature environment balances four pillars:
1. **Network**: The fabric and security (VNets, Firewalls).
2. **People**: RBAC, Identity, IAM.
3. **Process**: ITIL, Change management, Agile.
4. **Deployments**: CI/CD, Azure DevOps, GitHub Actions.

When writing code or deploying infrastructure, always consider the flow:
- **Intention**: What is the business goal?
- **Implementation**: What code/tool are we using?
- **Reason**: Why this specific tool/architecture over another?

---

## 1. Azure CLI (Command-Line Interface)
The Azure CLI is a cross-platform command-line tool (running on Windows, macOS, and Linux) used to create, manage, and automate Microsoft Azure resources. 
While the Azure Portal provides a visual interface, the CLI is essential for DevOps engineers because it allows you to integrate infrastructure management directly into your CI/CD pipelines (like GitHub Actions or Azure DevOps) and automate repetitive tasks via scripts.

### Command Structure
Commands follow a predictable, text-based pattern: `az [group] [subgroup] [action] [parameters]`

### Common Commands for Daily Operations
- `az login --use-device-code`: Log in securely from a remote VM or terminal.
- `az account list --output table`: View all your subscriptions in a readable format.
- `az account show`: See which subscription is currently active in your CLI context.
- `az group list --output table`: List all Resource Groups.
- `az group create --name <name> --location centralindia`: Create a new Resource Group.
- `az vm list --output table`: List your VMs and their status.
- `az vm image list --publisher Canonical --output table`: Find Linux images for deployment.
- `az network nsg rule list -g <rg> --nsg-name <nsg>`: View your open firewall ports.

---

## 2. Terraform (Infrastructure as Code)
Terraform is an Infrastructure as Code (IaC) tool created by HashiCorp. It allows you to define your cloud infrastructure (VNets, AKS clusters, Databases) in human-readable code files (`.tf`), and automatically provision them.

### Why use Terraform?
- **Multi-Cloud Support**: While ARM templates or Bicep only work for Azure, Terraform can provision resources across Azure, AWS, and GCP using the same HCL syntax.
- **State Management**: Terraform uses a `terraform.tfstate` file to keep track of the exact state of your resources. This allows it to know exactly what needs to be added, changed, or deleted without recreating the entire environment.
- **Reusability and Modularity**: You can use the exact same codebase to spin up `n` number of environments (e.g., `dev`, `staging`, `prod`) by simply passing different variable files and maintaining a separate statefile for each environment.
- **Authentication**: Terraform uses an **API key to communicate** with Azure. Specifically, you create a Service Principal in Entra ID and provide Terraform with the Client ID and Client Secret so it can authenticate autonomously without human intervention.

### The Core Terraform Workflow
1. `terraform init`
   - **When**: Run this first.
   - **What**: Downloads the required "provider" plugins (like the Azure provider) and initializes the backend where your state file will be stored.
2. `terraform plan`
   - **When**: Every time you change your `.tf` code.
   - **What**: A "dry run" that compares your code against the current state and shows you exactly what will be created, modified, or destroyed.
3. `terraform apply`
   - **When**: When the plan looks correct.
   - **What**: Executes the changes via the Azure API and updates your `terraform.tfstate` file.
4. `terraform destroy`
   - **When**: When you want to tear down an environment.
   - **Warning**: This will permanently delete everything defined in your code from Azure.

### Utility Commands
- `terraform fmt`: Automatically formats your `.tf` files to ensure proper spacing and indentation.
- `terraform validate`: Checks your code for syntax errors without needing to communicate with Azure.
- `terraform show`: Displays the current state of your infrastructure in a readable format.
- `terraform state list`: Shows every individual resource Terraform is currently tracking in its memory.
