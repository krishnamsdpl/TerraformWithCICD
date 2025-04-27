# TerraformWithCICD

## 1. Prerequisites
Ensure you have the following prerequisites:

- **Azure Account** with permissions to create resources.
- **Azure DevOps Account** with access to create pipelines and manage repositories.
- **Terraform** installed locally for testing your infrastructure code.
- **Service Principal** in Azure to authenticate Terraform.

## 2. Create Service Principal in Azure
Terraform needs access to your Azure subscription to create and manage resources. We’ll use a Service Principal (SP) for authentication.

1.Steps to create a Service Principal:

Open Azure Cloud Shell or use Azure CLI.

2.Run the following command to create a Service Principal and assign a role:
```Bash
az ad sp create-for-rbac --name "TerraformSP" --role contributor --scopes /subscriptions/{subscription_id}
```
This command will output JSON containing the details needed for Terraform to authenticate.

3.Save the appId, password, and tenant values from the output for later.

Service Principal:https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-service-principal-portal
Built in role RBAC: https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles

## 3. Set up Azure DevOps Project
 
 Go to Azure DevOps and create a new Project.

Within the project, you’ll need to:

Create a Git repository to store your Terraform code.

Set up Pipelines to automate the infrastructure deployment.

## 4. Create a Terraform Configuration
This configuration defines the Azure infrastructure you want to deploy.

Example main.tf to deploy a simple Azure resource (e.g., Virtual Network):

```hcl
provider "azurerm" {
  features {}
  client_id       = "YOUR-APP-ID"      # Service Principal Client ID
  client_secret   = "YOUR-SECRET"      # Service Principal Secret
  tenant_id       = "YOUR-TENANT-ID"   # Tenant ID
  subscription_id = "YOUR-SUB-ID"      # Subscription ID
}

resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = "East US"
}

resource "azurerm_virtual_network" "example" {
  name                = "example-vnet"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  address_space       = ["10.0.0.0/16"]
}
```
## 5. Push Terraform Code to Azure DevOps Repository

1. Initialize a Git repository in your local project folder (if not done already).

```bash
git init
git remote add origin https://dev.azure.com/{your_org}/{your_project}/_git/{your_repo}
git add .
git commit -m "Initial commit of Terraform code"
git push origin main
```
2. This code will now be stored in Azure DevOps Repository.

## 6. Create an Azure DevOps Pipeline for Terraform
Now, you need to automate the deployment of the infrastructure using an Azure DevOps Pipeline.

Steps:
Create a Pipeline in Azure DevOps:

- Go to Pipelines in Azure DevOps and click New Pipeline.
- Choose the repository where your Terraform code is stored.
- Select YAML as the pipeline type.
- Configure the Pipeline YAML file: In the root of your repository, create a file called azure-pipelines.yml. This file will define the pipeline.

Example of a pipeline for Terraform:

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: UseTerraform@0
  inputs:
    terraformVersion: '1.0.x'

- script: |
    terraform init
  displayName: 'Initialize Terraform'

- script: |
    terraform plan -out=tfplan
  displayName: 'Terraform Plan'

- script: |
    terraform apply -auto-approve tfplan
  displayName: 'Terraform Apply'

- script: |
    terraform output
  displayName: 'Terraform Output'

```
This YAML pipeline does the following:

- Triggers when code is pushed to the main branch.
- Uses the ubuntu-latest image to run the pipeline.
- Initializes Terraform, generates a plan, and applies it to deploy the resources.
  
3. Service Connection for Azure:

- In the Azure DevOps pipeline settings, configure an Azure service connection to authenticate using the Service Principal created earlier.
- Go to Project Settings > Service Connections > New Service Connection > Azure Resource Manager.
- Select Service Principal (automatic) and fill in the details from the Service Principal output (appId, password, tenantId, subscriptionId).


## 7. Run the Pipeline
Once the pipeline is set up, you can trigger it manually or automatically when changes are pushed to the repository.

- When triggered, the pipeline will:
1.Initialize Terraform to download provider plugins.
2.Run Terraform Plan to show the execution plan.
3.Apply the plan to deploy the infrastructure in Azure.
4.Output the result to show any relevant values.

## 8. Verify Infrastructure in Azure

After the pipeline successfully runs, you can verify that the resources were created by navigating to the Azure Portal and checking the Resource Group where the Virtual Network (or other resources) was created.

## 9. Terraform State Management
Since Terraform manages the state of infrastructure, it’s recommended to store the Terraform state file in a remote backend, such as Azure Storage, to prevent issues when working in teams.

You can configure a backend in your main.tf:

```yaml
terraform {
  backend "azurerm" {
    resource_group_name   = "rg-terraform"
    storage_account_name  = "tfstatestorage"
    container_name        = "tfstate"
    key                   = "terraform.tfstate"
  }
}
```
## 10. Optional: Managing Secrets and Variables
Use Azure Key Vault to store sensitive data such as service principal credentials or other secrets.
You can also define Terraform variables and pass them through the Azure DevOps Pipeline.

Example for using variables in main.tf:

```hcl
variable "location" {
  type    = string
  default = "East US"
}

resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = var.location
}
```
Pass variables from the pipeline:

```yaml
variables:
  location: "West US"

```

## 11. Monitor and Manage Pipeline Execution
- Monitor the pipeline execution in Azure DevOps.
- Use logs to debug issues during terraform plan and terraform apply.
- Set up automated alerts or additional steps for monitoring infrastructure after deployment.




