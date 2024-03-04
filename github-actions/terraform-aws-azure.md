# GitHub Actions workflow that manages infrastructure deployments using Terraform for AWS and Azure Resource Manager (ARM)

Below is an example GitHub Actions workflow that manages infrastructure deployments using Terraform for AWS and Azure Resource Manager (ARM) templates. It automatically applies changes to infrastructure when code changes are merged to certain branches:

```yaml
name: Infrastructure Deployment

on:
  push:
    branches:
      - main

jobs:
  terraform-aws:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Terraform for AWS
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.0.0
          cli_config_credentials_token: ${{ secrets.AWS_ACCESS_TOKEN }}

      - name: Terraform Init for AWS
        run: terraform init -backend-config="bucket=my-terraform-state-bucket" -backend-config="key=terraform.tfstate" -backend-config="region=us-east-1" ./aws

      - name: Terraform Plan for AWS
        run: terraform plan -out=tfplan ./aws

      - name: Terraform Apply for AWS
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan ./aws

  terraform-azure:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Terraform for Azure
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.0.0

      - name: Terraform Init for Azure
        run: terraform init -backend-config="resource_group_name=my-rg" -backend-config="storage_account_name=my-storage-account" -backend-config="container_name=tfstate" -backend-config="key=terraform.tfstate" ./azure

      - name: Terraform Plan for Azure
        run: terraform plan -out=tfplan ./azure

      - name: Terraform Apply for Azure
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve tfplan ./azure
```

This workflow consists of two jobs:

1. **terraform-aws**: Manages infrastructure deployment on AWS using Terraform.
   - It checks out the code, sets up Terraform, initializes Terraform with AWS backend configurations, creates an execution plan, and applies the changes if the workflow is triggered by a push to the main branch.

2. **terraform-azure**: Manages infrastructure deployment on Azure using Terraform.
   - It checks out the code, sets up Terraform, initializes Terraform with Azure backend configurations, creates an execution plan, and applies the changes if the workflow is triggered by a push to the main branch.

Ensure that you have the appropriate Terraform files (`*.tf`) for AWS and Azure in the respective directories (`./aws` and `./azure`) within your repository. Also, make sure to replace the placeholder values in backend configurations with your actual values.

Don't forget to configure GitHub Secrets to securely store any sensitive information required for your Terraform deployments, such as AWS access tokens or Azure service principals.
