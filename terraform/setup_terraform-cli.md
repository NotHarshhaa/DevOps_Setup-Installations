# ⚙️ Configuring Terraform CLI

## 1. 🌐 Set Up Environment Variables

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

## 2. 🗂️ Organize Configuration Directory

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

## 3. 🔄 Initialize Terraform Configuration

### Why?

Initializing a configuration sets up the backend and downloads the necessary provider plugins.

### How?

1. **Navigate to Your Project Directory:**

    ```bash
    cd ~/terraform-projects/my-project
    ```

2. **Run Init Command:**

    ```bash
    terraform init
    ```

## 4. ✏️ Create and Manage Variables

### Why?

Variables allow you to customize configurations dynamically.

### How?

1. **Create a Variables File:**

    ```hcl
    # variables.tf
    variable "region" {
      description = "The AWS region to deploy to"
      type        = string
      default     = "us-west-2"
    }
    ```

2. **Use Variables in Configuration:**

    ```hcl
    # main.tf
    provider "aws" {
      region = var.region
    }
    ```

3. **Set Variable Values:**
   - Inline:

     ```bash
     terraform apply -var="region=us-east-1"
     ```

   - Environment Variable:

     ```bash
     export TF_VAR_region="us-east-1"
     ```

## 5. 🛠️ Configure Backends

### Why?

Backends manage and store the state file remotely for collaboration and security.

### How?

1. **Example for AWS S3:**

    ```hcl
    # backend.tf
    terraform {
      backend "s3" {
        bucket         = "my-terraform-state"
        key            = "path/to/my/key"
        region         = "us-west-2"
        dynamodb_table = "terraform-lock"
      }
    }
    ```

2. **Reinitialize Terraform:**

    ```bash
    terraform init
    ```

## 6. 🔄 Enable Command Aliases

### Why?

Aliases simplify repeated command usage and can save time.

### How?

1. **Add Aliases to Profile:**

    ```bash
    echo 'alias tfinit="terraform init"' >> ~/.bashrc
    echo 'alias tfplan="terraform plan"' >> ~/.bashrc
    echo 'alias tfapply="terraform apply"' >> ~/.bashrc
    echo 'alias tfdestroy="terraform destroy"' >> ~/.bashrc
    ```

2. **Reload Profile:**

    ```bash
    source ~/.bashrc
    ```

## 7. 📦 Lock and Vendor Provider Versions

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
          version = "~> 3.0"
        }
      }
      required_version = ">= 0.14.0"
    }
    ```

2. **Reinitialize to Install Specified Versions:**

    ```bash
    terraform init -upgrade
    ```

## 8. 🔍 Format and Validate Code

### Why?

Formatting and validating ensure code consistency and catch errors early.

### How?

1. **Format Code:**

    ```bash
    terraform fmt
    ```

2. **Validate Configuration:**

    ```bash
    terraform validate
    ```

## 9. 📄 Generate and Use Terraform Docs

### Why?

Documentation helps in understanding and maintaining Terraform configurations.

### How?

1. **Install terraform-docs:**

    ```bash
    brew install terraform-docs
    ```

2. **Generate Documentation:**

    ```bash
    terraform-docs markdown table . > README.md
    ```

## 10. 📚 Additional Resources

- 📖 [Official Terraform CLI Documentation](https://www.terraform.io/docs/cli-index.html)
- 🎓 [HashiCorp Learn - Terraform CLI](https://learn.hashicorp.com/terraform?track=cli)

You are now ready to effectively configure and use the Terraform CLI! 🚀

## **Author by:**

![](https://imgur.com/2j6Aoyl.png)

> [!Note]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
