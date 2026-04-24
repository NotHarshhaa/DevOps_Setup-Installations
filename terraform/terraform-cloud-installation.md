# ☁️ Terraform Cloud Installation and Setup Guide

## 1. 🌍 What is Terraform Cloud?

Terraform Cloud is a managed service by HashiCorp that provides:
- **Remote state management** with built-in locking
- **Secure variable storage** for sensitive data
- **Collaborative workflows** with team access controls
- **Policy as code** with Sentinel and OPA
- **Cost estimation** and infrastructure insights
- **Private module registry** for reusable infrastructure code
- **Integration with VCS** (GitHub, GitLab, Bitbucket)

## 2. 📦 Prerequisites

Before setting up Terraform Cloud, ensure you have:
- **A valid email address** for sign-up
- **Terraform installed** locally (version 1.1.0 or later)
- **A version control system** account (GitHub, GitLab, or Bitbucket)
- **Cloud provider credentials** (AWS, Azure, GCP, etc.)

## 3. 🔄 Creating a Terraform Cloud Account

### Step 1: Sign Up

1. **Navigate to** [Terraform Cloud](https://app.terraform.io/signup)
2. **Choose your plan:**
   - **Free tier**: Up to 5 users, 500 runs/month, basic features
   - **Team tier**: More runs, SSO, advanced features
   - **Enterprise tier**: Unlimited runs, custom policies, dedicated support
3. **Complete the registration** with your email and password
4. **Verify your email address**

### Step 2: Create an Organization

1. **After login**, click on "Create an organization"
2. **Enter organization details:**
   - Organization name (e.g., `my-company-infrastructure`)
   - Email address
3. **Choose your email preference** for notifications
4. **Click "Create organization"**

## 4. 🏗️ Setting Up Workspaces

### Step 1: Create a Workspace

1. **Navigate to your organization** in Terraform Cloud
2. **Click "New workspace"**
3. **Choose workspace type:**
   - **CLI-driven**: For manual runs from local CLI
   - **VCS-driven**: For automatic runs on VCS changes
4. **Configure workspace settings:**
   - Workspace name (e.g., `production-networking`)
   - Description
   - Execution mode (local, remote, or agent)
   - Terraform version
5. **Click "Create workspace"**

### Step 2: Connect to VCS (for VCS-driven workspaces)

1. **Select "Connect to a version control provider"**
2. **Choose your VCS provider** (GitHub, GitLab, or Bitbucket)
3. **Authorize Terraform Cloud** to access your repositories
4. **Select the repository** containing your Terraform code
5. **Configure VCS settings:**
   - Branch to track (e.g., `main`)
   - Terraform working directory
   - Trigger conditions (push, PR, tags)

## 5. 🔐 Configuring Variables and Secrets

### Step 1: Add Environment Variables

1. **Navigate to your workspace** → "Variables"
2. **Click "Add variable"**
3. **Configure variable settings:**
   - Key: Variable name (e.g., `AWS_REGION`)
   - Value: Variable value
   - Category: Environment variable or Terraform variable
   - Sensitive: Mark as sensitive for secrets
   - HCL: Enable for complex values

**Example variables:**
```
AWS_ACCESS_KEY_ID (sensitive)
AWS_SECRET_ACCESS_KEY (sensitive)
AWS_REGION = us-west-2
ENVIRONMENT = production
```

### Step 2: Add Terraform Variables

1. **Create a `terraform.tfvars` file locally** for reference
2. **Add variables to Terraform Cloud workspace:**
   - Key: Variable name (matches your Terraform variables)
   - Value: Variable value
   - Category: Terraform variable

**Example:**
```
instance_type = t3.large
vpc_cidr = 10.0.0.0/16
enable_monitoring = true
```

## 6. 🔧 Configuring Terraform CLI for Cloud

### Step 1: Install Terraform CLI

Follow the installation guide in `install_terraform.md`

### Step 2: Authenticate with Terraform Cloud

**Option 1: Interactive Login**
```bash
terraform login
```
This will open a browser window for authentication.

**Option 2: API Token**
1. **Navigate to** User Settings → Tokens
2. **Generate a new token** with appropriate permissions
3. **Set the token as environment variable:**
```bash
export TF_API_TOKEN="your-api-token-here"
```

### Step 3: Configure Cloud Backend

Create or update `backend.tf`:
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

### Step 4: Initialize Terraform

```bash
terraform init
```

## 7. 🚀 Running Terraform in Cloud

### Step 1: Plan and Apply via CLI

```bash
# Plan changes
terraform plan

# Apply changes
terraform apply
```

### Step 2: Trigger Runs via VCS

1. **Push changes** to your connected branch
2. **Terraform Cloud** automatically triggers a run
3. **Monitor the run** in the Terraform Cloud dashboard
4. **Approve the run** if required (based on workspace settings)

### Step 3: Manual Runs in Cloud

1. **Navigate to your workspace** in Terraform Cloud
2. **Click "Queue plan"**
3. **Review the plan output**
4. **Click "Confirm & apply"** if changes look correct

## 8. 🔒 Setting Up Sentinel Policies

### Step 1: Enable Sentinel

1. **Navigate to Organization Settings** → "Sentinel"
2. **Ensure Sentinel is enabled** (available on paid plans)

### Step 2: Create a Policy Set

1. **Go to Organization** → "Policy Sets"
2. **Click "Create a new policy set"**
3. **Configure policy set:**
   - Name (e.g., `security-policies`)
   - Description
   - Workspace scope (specific workspaces or all)
   - Enforcement level (advisory, soft-mandatory, hard-mandatory)

### Step 3: Add Policies

1. **Create policy files** with `.sentinel` extension:

**Example policy:**
```sentinel
# restrict-s3-public-access.sentinel
import "tfplan/v2" as tfplan

# Restrict S3 bucket public access
s3_buckets_public = rule {
  all tfplan.resource_changes as rc {
    rc.type is "aws_s3_bucket" and
    rc.change.after.acl is "private"
  }
}

main = rule {
  s3_buckets_public
}
```

2. **Upload policies** to your policy set
3. **Assign policies** to workspaces

## 9. 📊 Monitoring and Notifications

### Step 1: Configure Notifications

1. **Navigate to workspace** → "Notifications"
2. **Add notification integrations:**
   - **Slack**: Connect to your Slack workspace
   - **Email**: Configure email recipients
   - **Microsoft Teams**: Add webhook URL
   - **Generic webhook**: Custom webhook endpoint

### Step 2: Monitor Runs

1. **View run history** in workspace dashboard
2. **Check run logs** for detailed execution information
3. **Review state changes** in the state view
4. **Track cost estimates** for infrastructure changes

## 10. 👥 Team Management

### Step 1: Invite Team Members

1. **Navigate to Organization** → "Teams"
2. **Create teams** (e.g., `developers`, `operators`, `auditors`)
3. **Add members** to teams via email
4. **Assign permissions** to teams

### Step 2: Configure Access Controls

**Workspace-level permissions:**
- **Read**: View runs and state
- **Plan**: Trigger plans
- **Write**: Trigger applies
- **Admin**: Full workspace control

**Organization-level permissions:**
- **Owner**: Full organization control
- **Admin**: Manage teams and policies
- **Member**: Basic access to assigned workspaces

## 11. 🔧 Using Terraform Cloud Agents

### Step 1: Install Terraform Cloud Agent

1. **Download the agent** from Terraform Cloud
2. **Install on your infrastructure** (on-premises or private cloud)

**Docker installation:**
```bash
docker pull hashicorp/tfc-agent:latest
docker run -d \
  -e TFC_AGENT_TOKEN="your-agent-token" \
  hashicorp/tfc-agent:latest
```

**Binary installation:**
```bash
wget https://releases.hashicorp.com/tfc-agent/<version>/tfc-agent_<version>_linux_amd64.zip
unzip tfc-agent_<version>_linux_amd64.zip
./tfc-agent
```

### Step 2: Configure Agent Pools

1. **Navigate to Organization** → "Agent Pools"
2. **Create an agent pool**
3. **Assign workspaces** to the agent pool
4. **Agents will execute runs** in your private network

## 12. 📦 Private Module Registry

### Step 1: Publish Modules

1. **Navigate to Organization** → "Registry"
2. **Click "Publish a module"**
3. **Connect to VCS repository** containing the module
4. **Configure module details:**
   - Module name
   - Provider (AWS, Azure, GCP, etc.)
   - Versioning strategy

### Step 2: Use Private Modules

Reference private modules in your Terraform code:
```hcl
module "vpc" {
  source  = "app.terraform.io/your-organization/vpc/aws"
  version = "1.0.0"

  cidr = "10.0.0.0/16"
}
```

## 13. 🔄 Migration from Local State

### Step 1: Backup Local State

```bash
cp terraform.tfstate terraform.tfstate.backup
```

### Step 2: Update Backend Configuration

Replace local backend with cloud backend in `backend.tf`:
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

### Step 3: Initialize and Migrate

```bash
terraform init
```
Terraform will prompt to migrate state to Terraform Cloud.

### Step 4: Verify Migration

1. **Check state in Terraform Cloud dashboard**
2. **Run `terraform plan`** to verify state integrity
3. **Remove local state file** after verification

## 14. 📚 Best Practices

### Security
- **Always mark sensitive variables** as sensitive
- **Use least privilege IAM roles** for cloud providers
- **Enable MFA** for Terraform Cloud accounts
- **Rotate API tokens regularly**
- **Use Sentinel policies** for compliance

### Collaboration
- **Use VCS-driven workspaces** for team projects
- **Require approval** for production applies
- **Set up notifications** for run status
- **Use workspace naming conventions** (e.g., `env-service`)
- **Document variable requirements** in README files

### Performance
- **Use remote execution** for large infrastructure
- **Implement state refresh** optimization
- **Use module caching** for faster runs
- **Enable cost estimation** to prevent overspending

## 15. 📚 Additional Resources

- 📖 [Official Terraform Cloud Documentation](https://www.terraform.io/docs/cloud/)
- 🎓 [HashiCorp Learn - Terraform Cloud](https://learn.hashicorp.com/collections/terraform/cloud)
- 📖 [Sentinel Policy Language](https://docs.hashicorp.com/sentinel)
- 📖 [Terraform Cloud API](https://www.terraform.io/docs/cloud/api/index.html)

You are now ready to use Terraform Cloud for your infrastructure management! 🚀

## **Author by:**

![](https://imgur.com/2j6Aoyl.png)

> [!Note]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
