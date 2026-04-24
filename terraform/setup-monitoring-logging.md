# 📊 Enabling Logging and Monitoring for Terraform Projects

## 1. 🌍 Why Enable Logging and Monitoring?

Enabling logging and monitoring for Terraform projects provides several benefits:

- **Visibility:** Gain insights into your infrastructure changes.
- **Troubleshooting:** Quickly identify and resolve issues.
- **Auditing:** Maintain a record of actions and changes for compliance and security.

## 2. 📦 Tools and Services

Popular tools and services for logging and monitoring include:

- **Terraform Cloud / Terraform Enterprise**
- **AWS CloudWatch**
- **Azure Monitor**
- **Google Cloud Logging**
- **ELK Stack (Elasticsearch, Logstash, Kibana)**
- **Prometheus and Grafana**

## 3. 🌐 Setting Up Logging and Monitoring with Terraform Cloud

### Step 1: 🔄 Create a Terraform Cloud Account

1. **Sign up** for a [Terraform Cloud account](https://app.terraform.io/signup).
2. **Create an organization** and a workspace for your project.

### Step 2: ⚙️ Configure Terraform Cloud

1. **Update your Terraform configuration** to use Terraform Cloud as the backend:

    ```hcl
    # backend.tf
    terraform {
      cloud {
        organization = "your-organization"
        workspaces {
          name = "your-workspace"
        }
      }
    }
    ```

2. **Enable workspace notifications** in Terraform Cloud settings:
   - Slack notifications
   - Email notifications
   - Webhook integrations

### Step 3: 🔐 Configure Secrets

1. **Navigate to your workspace settings** in Terraform Cloud.
2. **Add environment variables** for your provider credentials (e.g., `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).

### Step 4: ✅ Apply Configuration

1. **Run the following commands** to initialize and apply the configuration:

    ```bash
    terraform init
    terraform apply
    ```

### Step 5: 📊 Monitor Runs

1. **View run history** in Terraform Cloud dashboard
2. **Check run logs** for detailed execution information
3. **Set up cost estimation** to track infrastructure costs

## 4. 🌐 Setting Up Logging and Monitoring with AWS CloudWatch

### Step 1: 🔄 Configure AWS CloudWatch

1. **Create a CloudWatch log group** for your Terraform logs:

    ```hcl
    # cloudwatch.tf
    resource "aws_cloudwatch_log_group" "terraform" {
      name = "/aws/terraform"
      retention_in_days = 14
    }
    ```

### Step 2: ⚙️ Update Terraform Configuration

1. **Add a provider configuration** for CloudWatch:

    ```hcl
    # provider.tf
    provider "aws" {
      region = "us-west-2"
    }
    ```

2. **Configure Terraform to log to CloudWatch**:

    ```hcl
    # main.tf
    resource "aws_cloudwatch_log_stream" "terraform_log_stream" {
      name           = "terraform"
      log_group_name = aws_cloudwatch_log_group.terraform.name
    }

    resource "aws_iam_role" "terraform_logging" {
      name = "terraform-logging-role"

      assume_role_policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Action = "sts:AssumeRole"
            Effect = "Allow"
            Principal = {
              Service = "lambda.amazonaws.com"
            }
          }
        ]
      })
    }

    resource "aws_iam_role_policy" "terraform_logging_policy" {
      name = "terraform-logging-policy"
      role = aws_iam_role.terraform_logging.id

      policy = jsonencode({
        Version = "2012-10-17"
        Statement = [
          {
            Effect = "Allow"
            Action = [
              "logs:CreateLogGroup",
              "logs:CreateLogStream",
              "logs:PutLogEvents"
            ]
            Resource = "arn:aws:logs:*:*:*"
          }
        ]
      })
    }
    ```

3. **Enable Terraform logging**:

    ```bash
    export TF_LOG=INFO
    export TF_LOG_PATH=terraform.log
    ```

### Step 3: ✅ Apply Configuration

1. **Run the following commands** to initialize and apply the configuration:

    ```bash
    terraform init
    terraform apply
    ```

## 5. 🌐 Setting Up Logging and Monitoring with Google Cloud Logging

### Step 1: 🔄 Configure Google Cloud Logging

1. **Enable Google Cloud Logging** in your Google Cloud project:

    ```bash
    gcloud services enable logging.googleapis.com
    ```

### Step 2: ⚙️ Update Terraform Configuration

1. **Add a provider configuration** for Google Cloud:

    ```hcl
    # provider.tf
    provider "google" {
      project = "your-gcp-project"
      region  = "us-central1"
    }
    ```

2. **Configure Terraform to log to Google Cloud Logging**:

    ```hcl
    # main.tf
    resource "google_logging_project_sink" "terraform_sink" {
      name        = "terraform-sink"
      destination = "logging.googleapis.com/projects/your-gcp-project/locations/global/buckets/_Default"
      filter      = "resource.type=gce_instance"
    }
    ```

### Step 3: ✅ Apply Configuration

1. **Run the following commands** to initialize and apply the configuration:

    ```bash
    terraform init
    terraform apply
    ```

## 6. 🌐 Setting Up Logging and Monitoring with ELK Stack

### Step 1: 🔄 Install ELK Stack

1. **Set up Elasticsearch, Logstash, and Kibana** on your server. Follow the [official installation guide](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-elastic-stack.html).

### Step 2: ⚙️ Configure Logstash

1. **Create a Logstash configuration** file (e.g., `logstash.conf`):

    ```plaintext
    input {
      file {
        path => "/var/log/terraform/*.log"
        start_position => "beginning"
      }
    }

    output {
      elasticsearch {
        hosts => ["localhost:9200"]
        index => "terraform-logs"
      }
    }
    ```

2. **Start Logstash** with the configuration file:

    ```bash
    bin/logstash -f logstash.conf
    ```

### Step 3: ⚙️ Update Terraform Configuration

1. **Configure Terraform to log to the specified file**:

    ```hcl
    # main.tf
    resource "null_resource" "terraform_log" {
      triggers = {
        timestamp = "${timestamp()}"
      }

      provisioner "local-exec" {
        command = "echo 'Terraform apply completed' >> /var/log/terraform/terraform.log"
      }
    }
    ```

### Step 4: ✅ Apply Configuration

1. **Run the following commands** to initialize and apply the configuration:

    ```bash
    terraform init
    terraform apply
    ```

## 7. 🌐 Setting Up Monitoring with Prometheus and Grafana

### Step 1: 🔄 Install Prometheus and Grafana

1. **Set up Prometheus and Grafana** on your server. Follow the [official Prometheus installation guide](https://prometheus.io/docs/prometheus/latest/installation/) and the [Grafana installation guide](https://grafana.com/docs/grafana/latest/installation/).

### Step 2: ⚙️ Configure Prometheus

1. **Create a Prometheus configuration file** (e.g., `prometheus.yml`):

    ```yaml
    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: 'terraform'
        static_configs:
          - targets: ['localhost:9100']
    ```

2. **Start Prometheus** with the configuration file:

    ```bash
    ./prometheus --config.file=prometheus.yml
    ```

### Step 3: ⚙️ Configure Grafana

1. **Add Prometheus as a data source** in Grafana.
2. **Create dashboards** to visualize Terraform metrics.

### Step 4: ⚙️ Update Terraform Configuration

1. **Configure Terraform to expose metrics**:

    ```hcl
    # main.tf
    resource "null_resource" "terraform_metrics" {
      triggers = {
        timestamp = "${timestamp()}"
      }

      provisioner "local-exec" {
        command = "echo 'Terraform metrics updated' >> /var/log/terraform/metrics.log"
      }
    }
    ```

### Step 5: ✅ Apply Configuration

1. **Run the following commands** to initialize and apply the configuration:

    ```bash
    terraform init
    terraform apply
    ```

## 8. 🌐 Setting Up Monitoring with Datadog

### Step 1: 🔄 Install Datadog Agent

1. **Install the Datadog agent** on your infrastructure using Terraform:

    ```hcl
    # datadog.tf
    resource "aws_instance" "datadog_agent" {
      ami           = data.aws_ami.ubuntu.id
      instance_type = "t3.micro"

      user_data = <<-EOF
              #!/bin/bash
              DD_API_KEY="${var.datadog_api_key}"
              DD_SITE="datadoghq.com"
              bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"
          EOF
    }
    ```

### Step 2: ⚙️ Configure Datadog Integration

1. **Add Terraform integration** in Datadog dashboard
2. **Create custom metrics** for Terraform state changes
3. **Set up alerts** for infrastructure changes

### Step 3: 📊 Monitor Terraform Runs

1. **Create Datadog dashboards** for:
   - Terraform run duration
   - Resource creation/deletion rates
   - State file changes
   - Cost trends

## 9. 🔒 Using Sentinel Policies for Governance

### Step 1: 📝 Create Sentinel Policies

1. **Create a policy file** (e.g., `restrict-s3-bucket-policy.sentinel`):

    ```sentinel
    import "tfplan/v2" as tfplan

    # Restrict S3 bucket public access
    all_s3_buckets_public = rule {
        all tfplan.resource_changes as rc {
            rc.type is "aws_s3_bucket" and
            rc.change.after.acl is "private"
        }
    }

    main = rule {
        all_s3_buckets_public
    }
    ```

### Step 2: ⚙️ Apply Policies to Workspace

1. **Upload policies** to Terraform Cloud
2. **Assign policies** to workspaces
3. **Configure policy enforcement** (soft-mandatory, hard-mandatory, advisory)

## 10. 📚 Additional Resources

- 📖 [Official Terraform Documentation](https://www.terraform.io/docs/)
- 📖 [AWS CloudWatch Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
- 📖 [Google Cloud Logging Documentation](https://cloud.google.com/logging/docs)
- 📖 [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- 📖 [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- 📖 [Grafana Documentation](https://grafana.com/docs/)

You are now ready to enable logging and monitoring for your Terraform projects! 🚀

## **Author by:**

![](https://imgur.com/2j6Aoyl.png)

> [!Note]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
