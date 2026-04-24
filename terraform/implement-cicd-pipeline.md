# 🚀 Implementing a CI/CD Pipeline for Terraform Projects

## 1. 🌍 Why Implement CI/CD for Terraform?

Implementing CI/CD for Terraform projects ensures:

- **Consistency:** Automated tests and deployments guarantee consistent infrastructure changes.
- **Efficiency:** Automated processes save time and reduce manual errors.
- **Collaboration:** Teams can collaborate more effectively with automated workflows.

## 2. 🛠️ Prerequisites

Before setting up a CI/CD pipeline, ensure you have:

- **Version control system (e.g., Git)**
- **Terraform configuration files**
- **Access to a CI/CD service (e.g., GitHub Actions, GitLab CI/CD, Jenkins, CircleCI)**

## 3. 📦 Choosing a CI/CD Service

Popular CI/CD services include:

- **GitHub Actions**
- **GitLab CI/CD**
- **Jenkins**
- **CircleCI**

## 4. 🌐 Setting Up a CI/CD Pipeline with GitHub Actions

### Step 1: 🔄 Create a GitHub Repository

1. **Create a new repository** on GitHub.
2. **Clone the repository** to your local machine:

    ```bash
    git clone https://github.com/your-username/your-repo.git
    cd your-repo
    ```

### Step 2: 🗂️ Add Terraform Configuration Files

1. **Create your Terraform configuration files** (e.g., `main.tf`, `variables.tf`, `backend.tf`).
2. **Add and commit the files** to your repository:

    ```bash
    git add .
    git commit -m "Add Terraform configuration files"
    git push origin master
    ```

### Step 3: ⚙️ Create a GitHub Actions Workflow

1. **Create a `.github/workflows` directory** in your repository:

    ```bash
    mkdir -p .github/workflows
    ```

2. **Create a workflow file** (e.g., `terraform.yml`) in the `.github/workflows` directory:

    ```yaml
    # .github/workflows/terraform.yml
    name: Terraform CI/CD

    on:
      push:
        branches:
          - main
      pull_request:
        branches:
          - main
      workflow_dispatch:

    permissions:
      contents: read
      pull-requests: write

    jobs:
      terraform:
        name: Terraform Plan & Apply
        runs-on: ubuntu-latest

        steps:
        - name: Checkout code
          uses: actions/checkout@v4

        - name: Configure AWS Credentials
          uses: aws-actions/configure-aws-credentials@v4
          with:
            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            aws-region: us-west-2

        - name: Set up Terraform
          uses: hashicorp/setup-terraform@v3
          with:
            terraform_version: 1.7.0
            terraform_wrapper: false

        - name: Terraform Init
          run: terraform init

        - name: Terraform Format Check
          run: terraform fmt -check

        - name: Terraform Validate
          run: terraform validate

        - name: Terraform Security Scan
          uses: aquasecurity/tfsec-action@v0.1.0

        - name: Terraform Plan
          id: plan
          run: terraform plan -out=tfplan

        - name: Save Plan
          uses: actions/upload-artifact@v4
          with:
            name: tfplan
            path: tfplan

        - name: Comment PR with Plan
          uses: actions/github-script@v7
          if: github.event_name == 'pull_request'
          with:
            script: |
              const output = `#### Terraform Plan 📖\n\n\`\`\`\n${process.env.PLAN}\n\`\`\``;
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: output
              })
          env:
            PLAN: ${{ steps.plan.outputs.stdout }}

        - name: Terraform Apply
          if: github.ref == 'refs/heads/main' && github.event_name == 'push'
          run: terraform apply -auto-approve tfplan

      drift-detection:
        name: Drift Detection
        runs-on: ubuntu-latest
        if: github.event_name == 'schedule'

        steps:
        - name: Checkout code
          uses: actions/checkout@v4

        - name: Configure AWS Credentials
          uses: aws-actions/configure-aws-credentials@v4
          with:
            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            aws-region: us-west-2

        - name: Set up Terraform
          uses: hashicorp/setup-terraform@v3
          with:
            terraform_version: 1.7.0

        - name: Terraform Init
          run: terraform init

        - name: Terraform Plan
          run: terraform plan -detailed-exitcode

    env:
      TF_LOG: INFO
    ```

3. **Commit and push** the workflow file:

    ```bash
    git add .github/workflows/terraform.yml
    git commit -m "Add Terraform GitHub Actions workflow"
    git push origin main
    ```

### Step 4: 🔐 Configure Secrets

1. **Navigate to your repository settings** on GitHub.
2. **Add secrets** for your provider credentials (e.g., `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).

### Step 5: ✅ Trigger the Workflow

1. **Push a change** to the `main` branch or create a pull request.
2. **Monitor the workflow** on the GitHub Actions tab to ensure it runs successfully.

## 5. 🌐 Setting Up a CI/CD Pipeline with GitLab CI/CD

### Step 1: 🔄 Create a GitLab Repository

1. **Create a new project** on GitLab.
2. **Clone the repository** to your local machine:

    ```bash
    git clone https://gitlab.com/your-username/your-repo.git
    cd your-repo
    ```

### Step 2: 🗂️ Add Terraform Configuration Files

1. **Create your Terraform configuration files** (e.g., `main.tf`, `variables.tf`, `backend.tf`).
2. **Add and commit the files** to your repository:

    ```bash
    git add .
    git commit -m "Add Terraform configuration files"
    git push origin master
    ```

### Step 3: ⚙️ Create a GitLab CI/CD Pipeline

1. **Create a `.gitlab-ci.yml` file** in the root of your repository:

    ```yaml
    # .gitlab-ci.yml
    stages:
      - validate
      - plan
      - apply
      - destroy

    variables:
      TF_VERSION: "1.7.0"
      TF_ROOT: "${CI_PROJECT_DIR}"
      TF_IN_AUTOMATION: "true"

    before_script:
      - terraform --version
      - terraform init
      - terraform fmt -check

    validate:
      stage: validate
      script:
        - terraform validate
      only:
        - merge_requests
        - main

    plan:
      stage: plan
      script:
        - terraform plan -out=tfplan -json > tfplan.json
      artifacts:
        paths:
          - tfplan
          - tfplan.json
        expire_in: 1 week
      only:
        - merge_requests
        - main

    apply:
      stage: apply
      script:
        - terraform apply -auto-approve tfplan
      dependencies:
        - plan
      when: manual
      only:
        - main

    security-scan:
      stage: validate
      image:
        name: aquasec/tfsec-ci
        entrypoint: [""]
      script:
        - tfsec .
      allow_failure: true
      only:
        - merge_requests
        - main
    ```

2. **Commit and push** the pipeline configuration file:

    ```bash
    git add .gitlab-ci.yml
    git commit -m "Add GitLab CI/CD pipeline configuration"
    git push origin main
    ```

### Step 4: 🔐 Configure CI/CD Variables

1. **Navigate to your project settings** on GitLab.
2. **Add CI/CD variables** for your provider credentials (e.g., `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).

### Step 5: ✅ Trigger the Pipeline

1. **Push a change** to the `main` branch or create a merge request.
2. **Monitor the pipeline** on the GitLab CI/CD tab to ensure it runs successfully.

## 6. 🌐 Setting Up a CI/CD Pipeline with Jenkins

### Step 1: 🔄 Install Jenkins

1. **Download and install Jenkins** from the [official website](https://www.jenkins.io/).
2. **Set up Jenkins** and install necessary plugins (e.g., Git, Pipeline).

### Step 2: ⚙️ Create a Jenkins Pipeline

1. **Create a new pipeline job** in Jenkins.
2. **Configure the pipeline** with the following script:

    ```groovy
    pipeline {
        agent any

        environment {
            TF_VERSION = '1.7.0'
            AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
            AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        }

        stages {
            stage('Checkout') {
                steps {
                    checkout scm
                }
            }

            stage('Setup Terraform') {
                steps {
                    sh """
                    curl -LO https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip
                    unzip terraform_${TF_VERSION}_linux_amd64.zip
                    sudo mv terraform /usr/local/bin/
                    terraform --version
                    """
                }
            }

            stage('Terraform Init') {
                steps {
                    sh 'terraform init'
                }
            }

            stage('Terraform Format') {
                steps {
                    sh 'terraform fmt -check'
                }
            }

            stage('Terraform Validate') {
                steps {
                    sh 'terraform validate'
                }
            }

            stage('Terraform Plan') {
                steps {
                    sh 'terraform plan -out=tfplan'
                }
            }

            stage('Terraform Apply') {
                when {
                    branch 'main'
                }
                steps {
                    input message: 'Apply Terraform changes?', ok: 'Apply'
                    sh 'terraform apply -auto-approve tfplan'
                }
            }
        }

        post {
            always {
                cleanWs()
            }
        }
    }
    ```

### Step 3: 🔐 Configure Credentials

1. **Navigate to Jenkins credentials** under "Manage Jenkins".
2. **Add credentials** for your provider (e.g., AWS credentials).

### Step 4: ✅ Trigger the Pipeline

1. **Push a change** to the repository.
2. **Monitor the pipeline** in Jenkins to ensure it runs successfully.

## 7. 🔒 Security Best Practices for CI/CD

### Step 1: 🔐 Use OIDC Authentication

Instead of using long-lived credentials, use OIDC for authentication:

```yaml
# GitHub Actions with OIDC
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: us-west-2
```

### Step 2: 🔍 Implement Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_tfsec
      - id: terraform_checkov
```

### Step 3: 📊 Implement Cost Estimation

```yaml
- name: Terraform Cost Estimation
  uses: infracost/infracost-gh-action@v0.10.9
  with:
    path: '.'
  env:
    INFRACOST_API_KEY: ${{ secrets.INFRACOST_API_KEY }}
```

## 8. 📚 Additional Resources

- 📖 [Official Terraform Documentation](https://www.terraform.io/docs/)
- 📖 [GitHub Actions Documentation](https://docs.github.com/en/actions)
- 📖 [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- 📖 [Jenkins Documentation](https://www.jenkins.io/doc/)
- 📖 [CircleCI Documentation](https://circleci.com/docs/)

You are now ready to implement a CI/CD pipeline for your Terraform projects! 🚀

## **Author by:**

![](https://imgur.com/2j6Aoyl.png)

> [!Note]
> **Join Our** [Telegram Community](https://t.me/prodevopsguy) // [Follow me](https://github.com/NotHarshhaa) **for more DevOps & Cloud content.**
