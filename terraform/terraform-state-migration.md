# 🔄 Terraform State Migration Guide

## 1. 🌍 Why Migrate Terraform State?

Common reasons for state migration include:
- **Moving from local to remote backend** for team collaboration
- **Changing cloud providers** (S3 to GCS, etc.)
- **Upgrading to Terraform Cloud** for better management
- **Reorganizing infrastructure** with workspaces
- **Improving security** with encryption and access controls
- **Optimizing performance** with better backend configurations

## 2. 📦 Prerequisites

Before migrating state, ensure you have:
- **Backup of current state file**
- **Access to both source and destination backends**
- **Appropriate IAM permissions** for both backends
- **Terraform installed** (same version as used for state creation)
- **Downtime window** for production migrations

## 3. 🛡️ Pre-Migration Checklist

### Step 1: Backup Current State

```bash
# Pull current state
terraform state pull > terraform.tfstate.backup

# Copy state file
cp terraform.tfstate terraform.tfstate.pre-migration
```

### Step 2: Verify State Integrity

```bash
# Check state file
terraform show

# List managed resources
terraform state list

# Validate configuration
terraform validate
```

### Step 3: Document Current Configuration

```bash
# Save current backend configuration
cat backend.tf > backend.tf.backup

# Save current provider configuration
cat provider.tf > provider.tf.backup
```

### Step 4: Review Resource Dependencies

```bash
# Create a dependency graph
terraform graph > graph.dot

# Visualize dependencies (requires graphviz)
terraform graph | dot -Tpng > graph.png
```

## 4. 🚀 Migration Scenarios

### Scenario 1: Local to AWS S3 Backend

#### Step 1: Create S3 Bucket and DynamoDB Table

```bash
# Create S3 bucket
aws s3api create-bucket \
  --bucket my-terraform-state \
  --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-terraform-state \
  --versioning-configuration Status=Enabled

# Enable server-side encryption
aws s3api put-bucket-encryption \
  --bucket my-terraform-state \
  --server-side-encryption-configuration \
  '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        }
      }
    ]
  }'

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraform-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

#### Step 2: Update Backend Configuration

Create or update `backend.tf`:
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}
```

#### Step 3: Initialize New Backend

```bash
terraform init
```

Terraform will prompt to copy existing state to the new backend. Type `yes` to confirm.

#### Step 4: Verify Migration

```bash
# Check state in new backend
terraform state list

# Verify resources are managed
terraform show

# Test with a plan
terraform plan
```

### Scenario 2: AWS S3 to Google Cloud Storage

#### Step 1: Create GCS Bucket

```bash
# Create bucket
gsutil mb -l us-west1 gs://my-terraform-state

# Enable versioning
gsutil versioning set on gs://my-terraform-state

# Enable uniform bucket-level access
gsutil uniform bucket-level access set on gs://my-terraform-state
```

#### Step 2: Pull State from S3

```bash
# Pull state from S3
terraform state pull > terraform.tfstate
```

#### Step 3: Update Backend Configuration

Update `backend.tf`:
```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "terraform/state"
  }
}
```

#### Step 4: Initialize New Backend

```bash
terraform init
```

#### Step 5: Push State to GCS

```bash
terraform state push terraform.tfstate
```

#### Step 6: Verify Migration

```bash
terraform state list
terraform plan
```

### Scenario 3: Remote Backend to Terraform Cloud

#### Step 1: Create Terraform Cloud Workspace

1. **Sign in to** [Terraform Cloud](https://app.terraform.io)
2. **Navigate to your organization**
3. **Create a new workspace** (CLI-driven)
4. **Note the workspace name**

#### Step 2: Authenticate with Terraform Cloud

```bash
terraform login
```

#### Step 3: Update Backend Configuration

Update `backend.tf`:
```hcl
terraform {
  cloud {
    organization = "your-organization"

    workspaces {
      name = "your-workspace"
    }
  }
}
```

#### Step 4: Initialize Cloud Backend

```bash
terraform init
```

Terraform will prompt to migrate state. Type `yes` to confirm.

#### Step 5: Verify Migration

```bash
terraform state list
terraform plan
```

### Scenario 4: Migrating Between Workspaces

#### Step 1: Export State from Source Workspace

```bash
# Switch to source workspace
terraform workspace select dev

# Pull state
terraform state pull > dev-state.tfstate
```

#### Step 2: Switch to Target Workspace

```bash
# Create target workspace if needed
terraform workspace new prod

# Switch to target workspace
terraform workspace select prod
```

#### Step 3: Import State to Target Workspace

```bash
# Push state to new workspace
terraform state push dev-state.tfstate
```

#### Step 4: Verify Migration

```bash
terraform state list
terraform plan
```

### Scenario 5: Splitting State (State Partitioning)

#### Step 1: Identify Resources to Move

```bash
# List all resources
terraform state list

# Identify resources for new state file
terraform state list | grep "networking"
```

#### Step 2: Move Resources to New State

```bash
# Move specific resources
terraform state mv aws_vpc.main module.networking.aws_vpc.main
terraform state mv aws_subnet.private module.networking.aws_subnet.private
```

#### Step 3: Create Separate Configuration

Create new directory and configuration for moved resources:
```bash
mkdir networking
cd networking
```

Create `backend.tf` for new state:
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-west-2"
  }
}
```

#### Step 4: Initialize New State

```bash
terraform init
```

#### Step 5: Verify Both States

```bash
# Original directory
cd ..
terraform state list

# New directory
cd networking
terraform state list
```

## 5. 🔧 Advanced Migration Techniques

### Step 1: State Refactoring

Move resources within state without recreating them:

```bash
# Move resource to module
terraform state mv aws_instance.example module.app.aws_instance.example

# Rename resource
terraform state mv aws_instance.old_name aws_instance.new_name

# Move multiple resources
terraform state mv \
  'module.old_module.aws_instance.example' \
  'module.new_module.aws_instance.example'
```

### Step 2: Import Existing Resources

Import resources not currently managed by Terraform:

```bash
# Import single resource
terraform import aws_instance.example i-1234567890abcdef0

# Import module resource
terraform import module.vpc.aws_vpc.main vpc-12345678
```

### Step 3: State Drift Detection

Detect and fix state drift:

```bash
# Refresh state
terraform refresh

# Plan to detect drift
terraform plan

# If drift detected, update state
terraform apply -refresh-only
```

## 6. 🔄 Post-Migration Tasks

### Step 1: Clean Up Old State

```bash
# Remove old state file (after verification)
rm terraform.tfstate.backup
rm terraform.tfstate.pre-migration

# Remove old backend configuration
rm backend.tf.backup
```

### Step 2: Update CI/CD Pipelines

Update CI/CD configurations to use new backend:
- Update GitHub Actions workflows
- Update GitLab CI/CD pipelines
- Update Jenkins pipelines
- Update environment variables

### Step 3: Update Documentation

Update project documentation with:
- New backend configuration
- State file location
- Access controls and permissions
- Migration procedures

### Step 4: Monitor for Issues

Monitor infrastructure for:
- State lock issues
- Permission errors
- Performance degradation
- Resource drift

## 7. 🛡️ Rollback Procedures

### Step 1: Rollback to Previous Backend

If migration fails, rollback to previous state:

```bash
# Restore previous backend configuration
cp backend.tf.backup backend.tf

# Initialize with previous backend
terraform init

# Push backup state
terraform state push terraform.tfstate.backup
```

### Step 2: Restore from Backup

If state is corrupted:

```bash
# Restore from backup
cp terraform.tfstate.pre-migration terraform.tfstate

# Initialize backend
terraform init

# Verify state
terraform state list
```

### Step 3: Emergency Rollback

For critical failures:

```bash
# Stop all Terraform operations
# Restore infrastructure manually if needed
# Recreate state from infrastructure (terraform import)
```

## 8. 📊 Migration Best Practices

### Do's
- ✅ Always backup state before migration
- ✅ Test migration in non-production first
- ✅ Use version control for backend configuration
- ✅ Document migration steps
- ✅ Verify state integrity after migration
- ✅ Use appropriate IAM permissions
- ✅ Schedule migrations during maintenance windows

### Don'ts
- ❌ Migrate state without backup
- ❌ Migrate production state without testing
- ❌ Modify state file manually
- ❌ Ignore state lock errors
- ❌ Migrate during peak usage
- ❌ Skip verification steps

### Security Considerations
- Encrypt state at rest
- Use least privilege IAM roles
- Enable state locking
- Implement state versioning
- Audit state access logs
- Rotate encryption keys regularly

## 9. 🔍 Troubleshooting

### Issue 1: State Lock Timeout

**Problem:** State is locked and cannot be migrated

**Solution:**
```bash
# Force unlock state (use with caution)
terraform force-unlock <LOCK_ID>

# Or wait for lock to expire
```

### Issue 2: Permission Denied

**Problem:** Insufficient permissions to access backend

**Solution:**
- Verify IAM permissions
- Check bucket policies
- Ensure correct credentials
- Verify network access

### Issue 3: State Corruption

**Problem:** State file is corrupted

**Solution:**
```bash
# Restore from backup
terraform state push terraform.tfstate.backup

# Or rebuild state from infrastructure
terraform import <resource> <resource_id>
```

### Issue 4: Version Mismatch

**Problem:** Terraform version incompatibility

**Solution:**
```bash
# Check Terraform version
terraform --version

# Upgrade or downgrade Terraform as needed
tfenv install <version>
tfenv use <version>
```

## 10. 📚 Additional Resources

- 📖 [Official Terraform State Migration Documentation](https://www.terraform.io/docs/language/state/backends.html#migrating-state)
- 🎓 [HashiCorp Learn - State Management](https://learn.hashicorp.com/collections/terraform/state)
- 📖 [Terraform State Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/state.html)
- 📖 [Terraform Cloud State Migration](https://www.terraform.io/docs/cloud/migrate/index.html)

You are now ready to migrate Terraform state between different backends and configurations! 🚀

## **Author by:**

![](https://imgur.com/2j6Aoyl.png)

> [!Note]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
