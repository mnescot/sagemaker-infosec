# Multi-Cloud Authentication Guide

This guide provides step-by-step instructions for authenticating with Terraform and cloud provider CLIs to deploy the security operations platform across AWS, Azure, and GCP.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [AWS Authentication](#aws-authentication)
3. [Azure Authentication](#azure-authentication)
4. [Google Cloud Authentication](#google-cloud-authentication)
5. [Terraform Backend Configuration](#terraform-backend-configuration)
6. [Verification](#verification)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Ensure all CLIs are installed (already completed):
- ✅ Terraform v1.10.4+
- ✅ AWS CLI v2.33.6+
- ✅ Azure CLI v2.82.0+
- ✅ Google Cloud CLI v553.0.0+

---

## AWS Authentication

### Option 1: AWS SSO (Recommended for IAM Identity Center)

**Step 1: Configure SSO**
```bash
aws configure sso
```

You'll be prompted for:
- **SSO session name**: `my-sso-session` (or any name you prefer)
- **SSO start URL**: Your organization's SSO URL (e.g., `https://my-company.awsapps.com/start`)
- **SSO Region**: The region where your Identity Center is configured (e.g., `us-east-1`)
- **SSO registration scopes**: Press Enter for default (`sso:account:access`)

**Step 2: Authenticate via Browser**

A browser window will open. Sign in with your SSO credentials and approve the request.

**Step 3: Select Account and Role**
```
CLI will display available accounts - select the account number where you'll deploy
CLI will display available roles - select the role with appropriate permissions
```

**Step 4: Configure Default Settings**
- **Default region**: `us-east-1` (or your preferred region)
- **Default output format**: `json`

**Step 5: Set SSO Profile**
```bash
# Use the SSO profile for all AWS CLI commands
export AWS_PROFILE=<profile-name-from-sso-config>

# Or specify in each Terraform command
terraform plan -var="aws_profile=<profile-name>"
```

**Step 6: Refresh SSO Session (when expired)**
```bash
aws sso login --profile <profile-name>
```

### Option 2: Access Keys (Development Only)

**⚠️ WARNING**: Only use for development. Never commit credentials to version control.

**Step 1: Create Access Keys**
1. Go to AWS Console → IAM → Users → Your User → Security credentials
2. Create access key → Choose "Command Line Interface (CLI)"
3. Save the Access Key ID and Secret Access Key

**Step 2: Configure AWS CLI**
```bash
aws configure
```

Enter:
- **AWS Access Key ID**: `AKIAIOSFODNN7EXAMPLE`
- **AWS Secret Access Key**: `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`
- **Default region**: `us-east-1`
- **Default output format**: `json`

**Credentials Location**: `~/.aws/credentials` and `~/.aws/config`

### Option 3: IAM Roles (for EC2/ECS)

If running on AWS compute resources with an IAM role attached:

```bash
# No configuration needed - Terraform will automatically use the instance role
# Verify it works:
aws sts get-caller-identity
```

### Verify AWS Authentication

```bash
# Check who you're authenticated as
aws sts get-caller-identity

# Expected output:
# {
#     "UserId": "AIDAI...",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/your-user"
# }

# Test permissions
aws sts get-session-token
```

---

## Azure Authentication

### Option 1: Interactive Login (Recommended)

**Step 1: Login via Browser**
```bash
az login
```

A browser window will open. Sign in with your Azure credentials (work/school or Microsoft account).

**Step 2: Select Subscription**

If you have multiple subscriptions:
```bash
# List all subscriptions
az account list --output table

# Set the active subscription
az account set --subscription "<subscription-id or subscription-name>"
```

**Step 3: Verify Active Subscription**
```bash
az account show
```

### Option 2: Service Principal (CI/CD & Automation)

**Step 1: Create Service Principal**
```bash
# Create service principal with Contributor role at subscription scope
az ad sp create-for-rbac \
  --name "terraform-deployer" \
  --role Contributor \
  --scopes /subscriptions/<subscription-id>
```

**Expected Output**:
```json
{
  "appId": "12345678-1234-1234-1234-123456789012",
  "displayName": "terraform-deployer",
  "password": "your-client-secret",
  "tenant": "87654321-4321-4321-4321-210987654321"
}
```

**⚠️ IMPORTANT**: Save these values securely!

**Step 2: Set Environment Variables**
```bash
export ARM_CLIENT_ID="<appId>"
export ARM_CLIENT_SECRET="<password>"
export ARM_SUBSCRIPTION_ID="<subscription-id>"
export ARM_TENANT_ID="<tenant>"
```

**Step 3: Authenticate Using Service Principal**
```bash
az login --service-principal \
  -u $ARM_CLIENT_ID \
  -p $ARM_CLIENT_SECRET \
  --tenant $ARM_TENANT_ID
```

### Option 3: Managed Identity (for Azure VMs)

If running on Azure VM with managed identity:
```bash
az login --identity
```

### Verify Azure Authentication

```bash
# Check current account
az account show

# List all resource groups (tests read permissions)
az group list --output table

# Verify Terraform can authenticate
az account show --query "{subscription:id, tenant:tenantId}"
```

---

## Google Cloud Authentication

### Option 1: User Account (Recommended for Development)

**Step 1: Authenticate**
```bash
gcloud auth login
```

A browser window will open. Sign in with your Google account and grant permissions.

**Step 2: Set Default Project**
```bash
# List all projects
gcloud projects list

# Set active project
gcloud config set project <PROJECT_ID>
```

**Step 3: Enable Application Default Credentials (for Terraform)**
```bash
gcloud auth application-default login
```

This creates credentials that Terraform can use automatically.

**Credentials Location**: `~/.config/gcloud/application_default_credentials.json`

### Option 2: Service Account (CI/CD & Production)

**Step 1: Create Service Account**
```bash
# Create service account
gcloud iam service-accounts create terraform-deployer \
  --display-name "Terraform Deployer" \
  --description "Service account for Terraform deployments"
```

**Step 2: Grant Permissions**
```bash
# Get your project ID
PROJECT_ID=$(gcloud config get-value project)

# Grant Editor role (adjust as needed)
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:terraform-deployer@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/editor"

# Grant additional roles as needed
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:terraform-deployer@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"
```

**Step 3: Create and Download Key**
```bash
gcloud iam service-accounts keys create ~/gcp-terraform-key.json \
  --iam-account=terraform-deployer@${PROJECT_ID}.iam.gserviceaccount.com
```

**⚠️ IMPORTANT**: Keep this key file secure! Never commit to version control.

**Step 4: Set Environment Variable**
```bash
export GOOGLE_APPLICATION_CREDENTIALS=~/gcp-terraform-key.json
```

**Step 5: Activate Service Account (optional)**
```bash
gcloud auth activate-service-account \
  --key-file=~/gcp-terraform-key.json
```

### Option 3: Workload Identity (for GKE)

If running on GKE with Workload Identity configured:
```bash
# No additional configuration needed
# Terraform will automatically use the workload identity
```

### Verify GCP Authentication

```bash
# Check current configuration
gcloud config list

# Verify authenticated user/service account
gcloud auth list

# Test permissions
gcloud projects describe $(gcloud config get-value project)

# Verify Terraform can authenticate
gcloud auth application-default print-access-token
```

---

## Terraform Backend Configuration

For production deployments, configure remote state storage.

### AWS S3 Backend

**Step 1: Create S3 Bucket and DynamoDB Table**
```bash
# Create S3 bucket for state
aws s3api create-bucket \
  --bucket my-terraform-state-bucket \
  --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-terraform-state-bucket \
  --versioning-configuration Status=Enabled

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

**Step 2: Configure Backend in Terraform**

Create `terraform/backend.tf`:
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "sagemaker-infosec/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

### Azure Storage Backend

**Step 1: Create Storage Account and Container**
```bash
# Create resource group
az group create \
  --name terraform-state-rg \
  --location eastus

# Create storage account
az storage account create \
  --name mytfstatestorage \
  --resource-group terraform-state-rg \
  --location eastus \
  --sku Standard_LRS \
  --encryption-services blob

# Get storage account key
ACCOUNT_KEY=$(az storage account keys list \
  --resource-group terraform-state-rg \
  --account-name mytfstatestorage \
  --query '[0].value' -o tsv)

# Create blob container
az storage container create \
  --name tfstate \
  --account-name mytfstatestorage \
  --account-key $ACCOUNT_KEY
```

**Step 2: Configure Backend in Terraform**

Create `terraform-azure/backend.tf`:
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "mytfstatestorage"
    container_name       = "tfstate"
    key                  = "azure-ml-infosec.tfstate"
  }
}
```

**Step 3: Set Environment Variable**
```bash
export ARM_ACCESS_KEY=$ACCOUNT_KEY
```

### GCP Cloud Storage Backend

**Step 1: Create GCS Bucket**
```bash
# Create bucket for state
gsutil mb -p $(gcloud config get-value project) \
  -l us-east1 \
  gs://my-terraform-state-bucket/

# Enable versioning
gsutil versioning set on gs://my-terraform-state-bucket/
```

**Step 2: Configure Backend in Terraform**

Create `terraform-gcp/backend.tf`:
```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state-bucket"
    prefix = "vertex-ai-infosec"
  }
}
```

---

## Verification

Before deploying, verify all authentications:

```bash
# AWS
aws sts get-caller-identity

# Azure
az account show

# GCP
gcloud auth list
gcloud config list

# Terraform
terraform version
```

### Test Terraform Authentication

**AWS**:
```bash
cd terraform
terraform init
terraform plan -var="identity_center_instance_arn=arn:aws:sso:::instance/YOUR_INSTANCE" -var="aws_region=us-east-1"
```

**Azure**:
```bash
cd terraform-azure
terraform init
terraform plan
```

**GCP**:
```bash
cd terraform-gcp
terraform init
terraform plan
```

---

## Troubleshooting

### AWS Issues

**Problem**: `Unable to locate credentials`
```bash
# Solution: Check credentials file
cat ~/.aws/credentials

# Or re-run SSO login
aws sso login --profile <profile-name>

# Or set AWS_PROFILE
export AWS_PROFILE=<profile-name>
```

**Problem**: `Access Denied`
```bash
# Check current identity
aws sts get-caller-identity

# Verify permissions
aws iam get-user
```

### Azure Issues

**Problem**: `No subscriptions found`
```bash
# Solution: Re-login
az logout
az login

# Or check subscriptions
az account list
```

**Problem**: `Tenant not found`
```bash
# Login with specific tenant
az login --tenant <tenant-id>
```

**Problem**: Terraform can't authenticate
```bash
# Ensure service principal variables are set
echo $ARM_CLIENT_ID
echo $ARM_TENANT_ID
echo $ARM_SUBSCRIPTION_ID

# Re-export if needed
export ARM_CLIENT_ID="..."
export ARM_CLIENT_SECRET="..."
export ARM_SUBSCRIPTION_ID="..."
export ARM_TENANT_ID="..."
```

### GCP Issues

**Problem**: `Application Default Credentials not found`
```bash
# Solution: Create ADC
gcloud auth application-default login

# Or set service account key
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json
```

**Problem**: `Permission denied`
```bash
# Check current account
gcloud config list

# Check permissions
gcloud projects get-iam-policy $(gcloud config get-value project)
```

**Problem**: `Project not set`
```bash
# Set project
gcloud config set project <PROJECT_ID>

# Or in Terraform
terraform plan -var="project_id=<PROJECT_ID>"
```

### General Terraform Issues

**Problem**: `Backend initialization failed`
```bash
# Reconfigure backend
terraform init -reconfigure

# Or migrate state
terraform init -migrate-state
```

**Problem**: `Provider authentication failed`
```bash
# Remove cached provider plugins
rm -rf .terraform
terraform init
```

---

## Security Best Practices

### 1. Credential Storage
- ✅ Use SSO/OAuth where possible
- ✅ Store service principal/service account keys in secure locations
- ✅ Never commit credentials to version control
- ✅ Use environment variables for sensitive values
- ❌ Don't hardcode credentials in Terraform files

### 2. Access Control
- ✅ Use least privilege principle
- ✅ Create separate service accounts for CI/CD
- ✅ Rotate credentials regularly
- ✅ Enable MFA on all accounts
- ✅ Use role-based access (RBAC)

### 3. State File Security
- ✅ Enable encryption for state files
- ✅ Enable versioning on state storage
- ✅ Restrict access to state storage
- ✅ Use state locking to prevent concurrent modifications
- ❌ Don't store state files in public repositories

### 4. Environment Variables for CI/CD

For automated deployments, set these environment variables:

**AWS**:
```bash
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN  # If using temporary credentials
AWS_REGION
```

**Azure**:
```bash
ARM_CLIENT_ID
ARM_CLIENT_SECRET
ARM_SUBSCRIPTION_ID
ARM_TENANT_ID
```

**GCP**:
```bash
GOOGLE_APPLICATION_CREDENTIALS  # Path to service account key
GOOGLE_PROJECT                  # Project ID
```

---

## Quick Reference: Deployment Commands

### AWS SageMaker Deployment
```bash
cd terraform
terraform init
terraform plan -var="identity_center_instance_arn=arn:aws:sso:::instance/YOUR_INSTANCE" -var="aws_region=us-east-1"
terraform apply -var="identity_center_instance_arn=arn:aws:sso:::instance/YOUR_INSTANCE" -var="aws_region=us-east-1"
```

### Azure Machine Learning Deployment
```bash
cd terraform-azure
terraform init
terraform plan
terraform apply
```

### GCP Vertex AI Deployment
```bash
cd terraform-gcp
terraform init
terraform plan -var="project_id=YOUR_PROJECT_ID"
terraform apply -var="project_id=YOUR_PROJECT_ID"
```

---

## Next Steps

After authentication is configured:

1. Review the Terraform variables in each directory:
   - `terraform/variables.tf` (AWS)
   - `terraform-azure/variables.tf` (Azure)
   - `terraform-gcp/variables.tf` (GCP)

2. Create a `terraform.tfvars` file with your specific values:
   ```hcl
   # Example for AWS
   aws_region = "us-east-1"
   identity_center_instance_arn = "arn:aws:sso:::instance/YOUR_INSTANCE"
   vpc_id = "vpc-12345678"
   private_subnet_ids = ["subnet-12345678", "subnet-87654321"]
   allowed_cidr_blocks = ["10.0.0.0/8"]  # Your on-prem network
   ```

3. Initialize and deploy:
   ```bash
   terraform init
   terraform plan
   terraform apply
   ```

4. Follow the deployment guides:
   - [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md)
   - [MULTI_CLOUD_DEPLOYMENT_GUIDE.md](MULTI_CLOUD_DEPLOYMENT_GUIDE.md)
   - [DEPLOYMENT_ENVIRONMENT_SETUP.md](DEPLOYMENT_ENVIRONMENT_SETUP.md)

---

## Support

For issues with:
- **Terraform**: https://www.terraform.io/docs
- **AWS CLI**: https://docs.aws.amazon.com/cli/
- **Azure CLI**: https://learn.microsoft.com/cli/azure/
- **Google Cloud CLI**: https://cloud.google.com/sdk/docs
