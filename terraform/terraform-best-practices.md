# Terraform Best Practices and Advanced Configuration Guide

![banner](https://www.cloudbolt.io/wp-content/uploads/terraform-best-practicies-1024x654-1.png)

## 1. Project Structure and Organization

### 1.1. Standard Directory Layout
```plaintext
project-root/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
├── modules/
│   ├── networking/
│   ├── compute/
│   └── storage/
├── .gitignore
├── README.md
└── backend.tf
```

### 1.2. Module Structure Best Practices
```hcl
# modules/networking/main.tf
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  tags = merge(
    local.common_tags,
    {
      Name = "${local.prefix}-vpc"
    }
  )
}

output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}
```

## 2. State Management

### 2.1. Remote State Configuration (AWS S3 + DynamoDB)

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "environment/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-lock-table"
  }
}
```

### 2.2. State Locking Setup

```hcl
resource "aws_dynamodb_table" "terraform_lock" {
  name           = "terraform-lock-table"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name = "Terraform Lock Table"
  }
}
```

## 3. Variables and Data Types

### 3.1. Variable Definitions

```hcl
# variables.tf
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_types" {
  description = "Map of instance types per environment"
  type        = map(string)
  default     = {
    dev     = "t3.micro"
    staging = "t3.medium"
    prod    = "t3.large"
  }
}
```

### 3.2. Local Variables

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }

  prefix = "${var.project_name}-${var.environment}"
}
```

## 4. Resource Configuration

### 4.1. Resource Naming Convention

```hcl
resource "aws_instance" "web_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_types[var.environment]

  tags = merge(
    local.common_tags,
    {
      Name = "${local.prefix}-web-server"
    }
  )
}
```

### 4.2. Count vs For_each

```hcl
# Using for_each (Preferred)
resource "aws_iam_user" "developers" {
  for_each = toset(var.developer_usernames)
  name     = each.key
}

# Using count (Legacy)
resource "aws_iam_user" "developers" {
  count = length(var.developer_usernames)
  name  = var.developer_usernames[count.index]
}
```

## 5. Data Sources and Providers

### 5.1. Provider Configuration

```hcl
provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn = var.role_arn
  }

  default_tags {
    tags = local.common_tags
  }
}

provider "aws" {
  alias  = "us-east-1"
  region = "us-east-1"
}
```

### 5.2. Data Source Usage

```hcl
data "aws_vpc" "existing" {
  tags = {
    Environment = var.environment
  }
}

data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}
```

## 6. Module Usage

### 6.1. Module Versioning

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 3.0"

  name = "${local.prefix}-vpc"
  cidr = var.vpc_cidr

  azs             = var.availability_zones
  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs

  enable_nat_gateway = true
  single_nat_gateway = var.environment != "prod"

  tags = local.common_tags
}
```

### 6.2. Custom Module Example

```hcl
# modules/web-app/main.tf
module "alb" {
  source = "../alb"

  name               = "${local.prefix}-alb"
  vpc_id             = var.vpc_id
  subnets           = var.public_subnet_ids
  security_groups    = [aws_security_group.alb.id]
  target_group_name = "${local.prefix}-tg"
}

module "ecs" {
  source = "../ecs"

  cluster_name    = "${local.prefix}-cluster"
  container_image = var.container_image
  container_port  = var.container_port
  target_group_arn = module.alb.target_group_arn
}
```

## 7. Security Best Practices

### 7.1. Sensitive Data Handling

```hcl
# Using AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "db-password"
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}
```

### 7.2. IAM Policies

```hcl
resource "aws_iam_role" "service_role" {
  name = "${local.prefix}-service-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "service_role_policy" {
  role       = aws_iam_role.service_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role"
}
```

## 8. Testing and Validation

### 8.1. Terraform Validation

```hcl
# Custom validation rules
variable "allowed_ports" {
  type = list(number)
  validation {
    condition     = alltrue([for port in var.allowed_ports : port > 0 && port < 65536])
    error_message = "Ports must be between 1 and 65535."
  }
}
```

### 8.2. Testing with Terratest

```go
// test/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVPCCreation(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/vpc",
        Vars: map[string]interface{}{
            "environment": "test",
            "vpc_cidr": "10.0.0.0/16",
        },
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcID)
}
```

## 9. CI/CD Integration

### 9.1. GitHub Actions Workflow

```yaml
name: Terraform CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
      
    - name: Terraform Init
      run: terraform init
      
    - name: Terraform Format
      run: terraform fmt -check
      
    - name: Terraform Plan
      run: terraform plan -no-color
      
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply -auto-approve
```

## 10. Performance Optimization

### 10.1. Resource Optimization

```hcl
# Use data sources efficiently
data "aws_availability_zones" "available" {
  state = "available"
}

# Implement proper depends_on
resource "aws_instance" "app_server" {
  depends_on = [aws_db_instance.main]
}
```

### 10.2. State Performance

```hcl
# Implement state partitioning
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "environments/${terraform.workspace}/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-lock"
  }
}
```

## 11. Troubleshooting Guide

### 11.1. Common Issues
1. State Lock Issues
   ```bash
   # Force unlock state
   terraform force-unlock LOCK_ID
   ```

2. Plan/Apply Failures
   ```bash
   # Enable debug logging
   export TF_LOG=DEBUG
   export TF_LOG_PATH=terraform.log
   ```

3. Provider Authentication
   ```bash
   # Verify AWS credentials
   aws sts get-caller-identity
   ```

### 11.2. Best Practices for Debugging
- Use detailed error messages
- Check state file consistency
- Verify provider configurations
- Review resource dependencies
- Check for version conflicts

## 12. Documentation

### 12.1. Module Documentation

```hcl
/**
 * # Web Application Module
 *
 * This module creates a complete web application stack including:
 * - Application Load Balancer
 * - ECS Cluster
 * - Auto Scaling Group
 * - Security Groups
 *
 * ## Usage
 * ```hcl
 * module "web_app" {
 *   source = "./modules/web-app"
 *   
 *   app_name = "my-app"
 *   environment = "prod"
 *   vpc_id = module.vpc.vpc_id
 * }
 * ```
 */

variable "app_name" {
  description = "Name of the application"
  type        = string
}
``` 