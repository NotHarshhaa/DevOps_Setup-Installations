# 🖥️ Terraform CLI: Comprehensive Usage Guide

## 1. 🌍 What is Terraform CLI?

Terraform CLI is the command-line interface for Terraform, enabling you to:
- **Initialize** Terraform configurations
- **Plan** infrastructure changes
- **Apply** infrastructure changes
- **Manage** state files
- **Debug** and troubleshoot issues
- **Automate** infrastructure operations

## 2. 📦 Basic CLI Commands

### Step 1: Initialize Terraform

```bash
# Initialize a new or existing Terraform configuration
terraform init

# Initialize with specific backend configuration
terraform init -backend-config="backend.tfvars"

# Initialize without backend configuration
terraform init -backend=false

# Reconfigure backend (ignore existing state)
terraform init -reconfigure

# Upgrade modules and providers
terraform init -upgrade
```

### Step 2: Format Configuration

```bash
# Format all Terraform files in current directory
terraform fmt

# Format files recursively
terraform fmt -recursive

# Check if files are formatted (CI/CD friendly)
terraform fmt -check

# Display files that would be reformatted
terraform fmt -check -diff

# Format specific files
terraform fmt main.tf variables.tf
```

### Step 3: Validate Configuration

```bash
# Validate configuration syntax
terraform validate

# Validate with JSON output
terraform validate -json

# Validate specific directory
terraform validate ./modules/networking
```

### Step 4: Plan Changes

```bash
# Generate and show execution plan
terraform plan

# Save plan to file
terraform plan -out=tfplan

# Plan with specific variable file
terraform plan -var-file="prod.tfvars"

# Plan with specific variables
terraform plan -var="instance_type=t3.large"

# Plan without refreshing state
terraform plan -refresh=false

# Plan with detailed exit code (for CI/CD)
terraform plan -detailed-exitcode

# Plan with lock timeout
terraform plan -lock-timeout=5m

# Plan in compact mode
terraform plan -compact
```

### Step 5: Apply Changes

```bash
# Apply changes with auto-approval
terraform apply -auto-approve

# Apply saved plan
terraform apply tfplan

# Apply with specific variables
terraform apply -var="environment=prod"

# Apply with variable file
terraform apply -var-file="staging.tfvars"

# Apply without refreshing state
terraform apply -refresh=false

# Apply with lock timeout
terraform apply -lock-timeout=10m

# Apply with parallelism limit
terraform apply -parallelism=2
```

## 3. 🔄 State Management Commands

### Step 1: View State

```bash
# Show all resources in state
terraform state list

# Show detailed state information
terraform show

# Show specific resource in state
terraform show aws_instance.example

# Pull state to file
terraform state pull > terraform.tfstate.backup

# Push state from file
terraform state push terraform.tfstate.backup
```

### Step 2: Move Resources in State

```bash
# Move resource within state
terraform state mv aws_instance.old aws_instance.new

# Move resource to module
terraform state mv aws_vpc.main module.networking.aws_vpc.main

# Move resources between modules
terraform state mv module.old.module.aws_instance.example module.new.module.aws_instance.example

# Move with state backup
terraform state mv -state-out=backup.tfstate aws_instance.old aws_instance.new
```

### Step 3: Remove Resources from State

```bash
# Remove resource from state (not from infrastructure)
terraform state rm aws_instance.example

# Remove multiple resources
terraform state rm aws_instance.example1 aws_instance.example2

# Remove module from state
terraform state rm module.vpc
```

### Step 4: Import Resources

```bash
# Import existing resource into state
terraform import aws_instance.example i-1234567890abcdef0

# Import module resource
terraform import module.vpc.aws_vpc.main vpc-12345678

# Import with provider configuration
terraform import -provider=registry.terraform.io/hashicorp/aws aws_instance.example i-1234567890abcdef0
```

## 4. 🏗️ Workspace Commands

### Step 1: List Workspaces

```bash
# List all workspaces
terraform workspace list

# Show current workspace
terraform workspace show
```

### Step 2: Create and Switch Workspaces

```bash
# Create new workspace
terraform workspace new dev

# Switch to existing workspace
terraform workspace select prod

# Delete workspace
terraform workspace delete staging
```

## 5. 📊 Output Commands

### Step 1: View Outputs

```bash
# Show all outputs
terraform output

# Show specific output
terraform output vpc_id

# Show output in JSON format
terraform output -json

# Show output in raw format
terraform output -raw vpc_id
```

## 6. 🔧 Provider Commands

### Step 1: Provider Operations

```bash
# Show installed providers
terraform providers

# Show provider schema
terraform providers schema

# Show provider schema in JSON
terraform providers schema -json

# Lock provider versions
terraform providers lock -platform=linux_amd64 -platform=darwin_amd64
```

## 7. 🧪 Test Commands (Terraform 1.6+)

### Step 1: Run Tests

```bash
# Run all tests
terraform test

# Run tests with verbose output
terraform test -v

# Run specific test file
terraform test tests/main_test.tftest.hcl

# Run tests with filtering
terraform test -filter="test_vpc"

# Run tests without updating state
terraform test -no-color
```

## 8. 🐛 Debugging Commands

### Step 1: Enable Debug Logging

```bash
# Set log level to DEBUG
export TF_LOG=DEBUG

# Set log level to TRACE
export TF_LOG=TRACE

# Set log level to INFO
export TF_LOG=INFO

# Set log level to WARN
export TF_LOG=WARN

# Set log level to ERROR
export TF_LOG=ERROR

# Disable logging
export TF_LOG=""

# Log to file
export TF_LOG_PATH=terraform.log

# Log with timestamp
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform.log
```

## 9. 🌐 Set Up Environment Variables

### Why?

Environment variables help manage sensitive information and configuration settings required by Terraform.

### How?

1. **AWS Example:**

    ```bash
    export AWS_ACCESS_KEY_ID="your-access-key-id"
    export AWS_SECRET_ACCESS_KEY="your-secret-access-key"
    export AWS_DEFAULT_REGION="us-west-2"
    ```

2. **Add to Profile:**
   To make these variables persistent across sessions, add them to your shell profile file (`.bashrc`, `.zshrc`, etc.):

    ```bash
    echo 'export AWS_ACCESS_KEY_ID="your-access-key-id"' >> ~/.bashrc
    echo 'export AWS_SECRET_ACCESS_KEY="your-secret-access-key"' >> ~/.bashrc
    echo 'export AWS_DEFAULT_REGION="us-west-2"' >> ~/.bashrc
    ```

3. **Terraform-Specific Variables:**

    ```bash
    # Set Terraform log level
    export TF_LOG=DEBUG

    # Set Terraform log path
    export TF_LOG_PATH=terraform.log

    # Set Terraform data directory
    export TF_DATA_DIR="/custom/terraform/directory"

    # Set plugin cache directory
    export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"

    # Disable interactive input
    export TF_INPUT=0

    # Set automation mode
    export TF_IN_AUTOMATION=1

    # Set Terraform Cloud token
    export TF_API_TOKEN="your-api-token"
    ```

## 10. 🗂️ Organize Configuration Directory

### Why?

Organizing your configuration files helps maintain clarity and structure in your Terraform projects.

### How?

1. **Create a Base Directory:**

    ```bash
    mkdir -p ~/terraform-projects/my-project
    cd ~/terraform-projects/my-project
    ```

2. **Subdirectories:**
   Create subdirectories for different environments (e.g., dev, staging, prod):

    ```bash
    mkdir dev staging prod
    ```

## 11. 🔄 Enable Command Aliases

### Why?

Aliases simplify repeated command usage and can save time.

### How?

1. **Add Aliases to Profile:**

    ```bash
    echo 'alias tf="terraform"' >> ~/.bashrc
    echo 'alias tfi="terraform init"' >> ~/.bashrc
    echo 'alias tfp="terraform plan"' >> ~/.bashrc
    echo 'alias tfa="terraform apply"' >> ~/.bashrc
    echo 'alias tfaa="terraform apply -auto-approve"' >> ~/.bashrc
    echo 'alias tfv="terraform validate"' >> ~/.bashrc
    echo 'alias tff="terraform fmt"' >> ~/.bashrc
    echo 'alias tfs="terraform state list"' >> ~/.bashrc
    echo 'alias tfsh="terraform show"' >> ~/.bashrc
    echo 'alias tfo="terraform output"' >> ~/.bashrc
    echo 'alias tfoj="terraform output -json"' >> ~/.bashrc
    ```

2. **Reload Profile:**

    ```bash
    source ~/.bashrc
    ```

## 12. 📦 Lock and Vendor Provider Versions

### Why?

Locking provider versions ensures consistent and reproducible deployments.

### How?

1. **Specify Provider Versions:**

    ```hcl
    # versions.tf
    terraform {
      required_providers {
        aws = {
          source  = "hashicorp/aws"
          version = "~> 5.0"
        }
        azurerm = {
          source  = "hashicorp/azurerm"
          version = "~> 3.0"
        }
      }
      required_version = ">= 1.0.0"
    }
    ```

2. **Reinitialize to Install Specified Versions:**

    ```bash
    terraform init -upgrade
    ```

3. **Lock Provider for Multiple Platforms:**

    ```bash
    terraform providers lock -platform=linux_amd64 -platform=darwin_amd64 -platform=windows_amd64
    ```

## 13. 📄 Generate and Use Terraform Docs

### Why?

Documentation helps in understanding and maintaining Terraform configurations.

### How?

1. **Install terraform-docs:**

    ```bash
    # macOS
    brew install terraform-docs

    # Linux
    curl -Lo ./terraform-docs https://github.com/terraform-docs/terraform-docs/releases/download/v0.16.0/terraform-docs-v0.16.0-$(uname)-amd64
    chmod +x terraform-docs
    sudo mv terraform-docs /usr/local/bin/
    ```

2. **Generate Documentation:**

    ```bash
    # Generate markdown table
    terraform-docs markdown table . > README.md

    # Generate markdown document
    terraform-docs markdown document . > README.md

    # Generate JSON
    terraform-docs json . > docs.json
    ```

## 14. 🚀 CI/CD Usage Examples

### Step 1: GitHub Actions

```yaml
- name: Terraform Init
  run: terraform init

- name: Terraform Validate
  run: terraform validate

- name: Terraform Plan
  run: terraform plan -out=tfplan

- name: Terraform Apply
  run: terraform apply -auto-approve tfplan
```

### Step 2: GitLab CI/CD

```yaml
validate:
  script:
    - terraform init
    - terraform validate

plan:
  script:
    - terraform init
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan

apply:
  script:
    - terraform apply -auto-approve tfplan
```

## 15. 📚 Best Practices

### Do's
- ✅ Always run `terraform plan` before `terraform apply`
- ✅ Use `-out` flag with plan and apply the saved plan
- ✅ Commit `.terraform.lock.hcl` for provider version consistency
- ✅ Use variable files for environment-specific configuration
- ✅ Enable debug logging when troubleshooting
- ✅ Use `terraform fmt -check` in CI/CD pipelines
- ✅ Lock provider versions in `versions.tf`

### Don'ts
- ❌ Use `terraform apply -auto-approve` in production
- ❌ Commit `.tfstate` files to version control
- ❌ Ignore state lock errors
- ❌ Use sensitive data in plain text variables
- ❌ Run terraform as root unless necessary
- ❌ Ignore validation errors

## 16. 📚 Additional Resources

- 📖 [Official Terraform CLI Documentation](https://www.terraform.io/cli/commands)
- 🎓 [HashiCorp Learn - Terraform CLI](https://learn.hashicorp.com/tutorials/terraform/terraform-cli)
- 📖 [Terraform Configuration Syntax](https://www.terraform.io/docs/language/syntax/configuration.html)

You are now ready to use the Terraform CLI effectively! 🚀

## **Author by:**

![](https://imgur.com/2j6Aoyl.png)

> [!Note]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
