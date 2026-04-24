# 🗂️ Terraform Workspaces: Managing Multiple Environments

## 1. 🌍 What are Terraform Workspaces?

Terraform workspaces allow you to manage multiple distinct but similar infrastructure configurations within a single Terraform configuration. They are particularly useful for:

- **Environment separation** (dev, staging, production)
- **Multi-region deployments**
- **Multi-tenant infrastructure**
- **Testing infrastructure changes**

**Important Note:** Workspaces are not a complete isolation mechanism. For true environment isolation, consider using separate directories or Terraform Cloud workspaces.

## 2. 📦 Prerequisites

- Terraform installed (version 0.12 or later)
- Basic understanding of Terraform configuration
- Cloud provider credentials configured

## 3. 🏗️ Setting Up Workspaces

### Step 1: Check Current Workspace

```bash
terraform workspace show
```

Default workspace is named `default`.

### Step 2: List All Workspaces

```bash
terraform workspace list
```

### Step 3: Create a New Workspace

```bash
terraform workspace new dev
```

This switches to the new workspace automatically.

### Step 4: Switch Between Workspaces

```bash
terraform workspace switch prod
```

Or use the old syntax:
```bash
terraform workspace select prod
```

### Step 5: Delete a Workspace

```bash
terraform workspace delete dev
```

**Note:** You cannot delete the currently active workspace or the `default` workspace.

## 4. ⚙️ Configuring Workspaces in Code

### Step 1: Use Workspace Name in Configuration

Reference the workspace name using `terraform.workspace`:

```hcl
# main.tf
provider "aws" {
  region = var.aws_region
}

resource "aws_instance" "app_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  tags = {
    Name        = "${terraform.workspace}-app-server"
    Environment = terraform.workspace
  }
}
```

### Step 2: Workspace-Specific Variables

Create conditional logic based on workspace:

```hcl
# variables.tf
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

# main.tf
locals {
  instance_type = {
    default = "t2.micro"
    dev     = "t3.micro"
    staging = "t3.medium"
    prod    = "t3.large"
  }
}

resource "aws_instance" "app_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type[terraform.workspace]
}
```

### Step 3: Workspace-Specific Backend Configuration

Configure backend to use workspace-specific state files:

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "${terraform.workspace}/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

## 5. 🌐 Environment-Specific Configurations

### Step 1: Directory Structure for Workspaces

```
project-root/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
├── terraform.tfvars
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars
```

### Step 2: Environment-Specific Variable Files

Create variable files for each environment:

**environments/dev.tfvars:**
```hcl
aws_region     = "us-west-2"
instance_type  = "t3.micro"
enable_monitoring = false
min_size       = 1
max_size       = 2
```

**environments/prod.tfvars:**
```hcl
aws_region     = "us-east-1"
instance_type  = "t3.large"
enable_monitoring = true
min_size       = 3
max_size       = 10
```

### Step 3: Load Environment-Specific Variables

```bash
terraform workspace select prod
terraform apply -var-file="environments/prod.tfvars"
```

## 6. 🔧 Advanced Workspace Patterns

### Step 1: Workspace Validation

Add validation to ensure only valid workspaces are used:

```hcl
# variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], terraform.workspace)
    error_message = "Invalid workspace. Must be dev, staging, or prod."
  }
}
```

### Step 2: Workspace-Specific Resource Counting

```hcl
# main.tf
locals {
  instance_count = {
    default = 1
    dev     = 1
    staging = 2
    prod    = 3
  }
}

resource "aws_instance" "app_server" {
  count         = local.instance_count[terraform.workspace]
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type[terraform.workspace]

  tags = {
    Name = "${terraform.workspace}-app-server-${count.index}"
  }
}
```

### Step 3: Workspace-Specific Module Configuration

```hcl
module "networking" {
  source = "./modules/networking"

  vpc_cidr = {
    default = "10.0.0.0/16"
    dev     = "10.1.0.0/16"
    staging = "10.2.0.0/16"
    prod    = "10.3.0.0/16"
  }[terraform.workspace]

  environment = terraform.workspace
}
```

## 7. 🔄 Managing State Across Workspaces

### Step 1: Initialize Backend for Each Workspace

```bash
# Switch to workspace
terraform workspace select dev

# Initialize backend
terraform init
```

### Step 2: Verify State Isolation

```bash
# Check state for current workspace
terraform show

# Switch workspace and verify different state
terraform workspace select prod
terraform show
```

### Step 3: State Management Best Practices

- **Use remote backends** (S3, GCS, Azure Blob) for workspace state
- **Enable state locking** with DynamoDB or equivalent
- **Enable state encryption** at rest
- **Use state versioning** for rollback capability

## 8. 🚀 CI/CD Integration with Workspaces

### Step 1: GitHub Actions Example

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        workspace: [dev, staging, prod]

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.7.0

    - name: Terraform Init
      run: terraform init

    - name: Terraform Workspace Select
      run: terraform workspace select ${{ matrix.workspace }}

    - name: Terraform Plan
      run: terraform plan -out=tfplan

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && matrix.workspace == 'dev'
      run: terraform apply -auto-approve tfplan
```

### Step 2: GitLab CI/CD Example

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - plan
  - apply

variables:
  TF_VERSION: "1.7.0"

.terraform_template: &terraform_template
  before_script:
    - terraform --version
    - terraform init
  image:
    name: hashicorp/terraform:${TF_VERSION}
    entrypoint: [""]

validate:
  <<: *terraform_template
  stage: validate
  script:
    - terraform validate

plan_dev:
  <<: *terraform_template
  stage: plan
  script:
    - terraform workspace select dev
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan

plan_prod:
  <<: *terraform_template
  stage: plan
  script:
    - terraform workspace select prod
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan
  when: manual

apply_dev:
  <<: *terraform_template
  stage: apply
  script:
    - terraform workspace select dev
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan_dev

apply_prod:
  <<: *terraform_template
  stage: apply
  script:
    - terraform workspace select prod
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan_prod
  when: manual
```

## 9. 🔒 Security Considerations

### Step 1: Workspace Isolation

**Limitations:**
- Workspaces share the same configuration code
- Some resources might have naming conflicts
- State files are stored in the same backend (unless configured otherwise)

**Best Practices:**
- Use unique resource names per workspace
- Implement workspace-specific IAM roles
- Use separate backend buckets for different security levels
- Consider Terraform Cloud workspaces for better isolation

### Step 2: Workspace-Specific IAM Roles

```hcl
# main.tf
data "aws_iam_role" "workspace_role" {
  name = "${terraform.workspace}-terraform-role"
}

provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn = data.aws_iam_role.workspace_role.arn
  }
}
```

## 10. 📊 Monitoring and Auditing

### Step 1: Workspace-Specific Tags

```hcl
locals {
  common_tags = {
    Environment = terraform.workspace
    ManagedBy   = "Terraform"
    Project     = var.project_name
    Workspace   = terraform.workspace
  }
}

resource "aws_instance" "app_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  tags = local.common_tags
}
```

### Step 2: Workspace-Specific CloudWatch Metrics

```hcl
resource "aws_cloudwatch_log_group" "app_logs" {
  name              = "/aws/${terraform.workspace}/app-logs"
  retention_in_days = terraform.workspace == "prod" ? 30 : 7
}
```

## 11. 🔄 Workspace Migration

### Step 1: Migrate from Single to Multi-Workspace

1. **Backup existing state:**
```bash
terraform state pull > terraform.tfstate.backup
```

2. **Create new workspace:**
```bash
terraform workspace new prod
```

3. **Initialize backend:**
```bash
terraform init
```

4. **Verify state:**
```bash
terraform state list
```

### Step 2: Import Resources to New Workspace

```bash
# Switch to target workspace
terraform workspace select dev

# Import existing resources
terraform import aws_instance.example i-1234567890abcdef0
```

## 12. 📚 Best Practices

### Do's
- ✅ Use workspaces for similar infrastructure across environments
- ✅ Implement workspace-specific validation
- ✅ Use remote backends with workspace-specific state paths
- ✅ Document workspace conventions in your README
- ✅ Use workspace names in resource tags for identification

### Don'ts
- ❌ Use workspaces for completely different infrastructure
- ❌ Store sensitive data in workspace names
- ❌ Assume workspaces provide complete isolation
- ❌ Use the default workspace for production
- ❌ Mix workspace management with directory-based separation

### Naming Conventions
- Use lowercase names: `dev`, `staging`, `prod`
- Use hyphens for multi-word names: `us-east-1`, `team-alpha`
- Avoid spaces and special characters
- Be consistent across projects

## 13. 🆚 Workspaces vs. Terraform Cloud Workspaces

### Terraform CLI Workspaces
- **Free and built-in** to Terraform CLI
- **State management** only
- **Limited isolation** capabilities
- **Manual configuration** required

### Terraform Cloud Workspaces
- **Managed service** with additional features
- **Full isolation** with separate state and variables
- **Built-in CI/CD** and collaboration features
- **Policy enforcement** with Sentinel
- **Cost estimation** and insights

**Recommendation:** Use Terraform Cloud workspaces for team collaboration and production environments.

## 14. 📚 Additional Resources

- 📖 [Official Terraform Workspaces Documentation](https://www.terraform.io/docs/language/state/workspaces.html)
- 🎓 [HashiCorp Learn - Workspaces](https://learn.hashicorp.com/tutorials/terraform/workspace)
- 📖 [Terraform Cloud Workspaces](https://www.terraform.io/docs/cloud/workspaces/index.html)

You are now ready to use Terraform workspaces for managing multiple environments! 🚀

## **Author by:**

![](https://imgur.com/2j6Aoyl.png)

> [!Note]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
