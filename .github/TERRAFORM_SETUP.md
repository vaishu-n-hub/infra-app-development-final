# Terraform GitHub Actions Setup Guide

## Overview
This workflow implements enterprise-grade Terraform deployment with manual approval gates for production changes.

## Key Features

✅ **Automatic validation and planning** on PRs  
✅ **Manual approval required** for terraform apply  
✅ **OIDC authentication** with AWS (no long-lived credentials)  
✅ **Remote state management** with S3 + DynamoDB locking  
✅ **Plan artifacts** preserved between jobs  
✅ **PR comments** with plan output  
✅ **Environment protection** rules

---

## Prerequisites

### 1. AWS Resources Setup

Create the following AWS resources for Terraform state management:

```bash
# S3 bucket for state storage
aws s3api create-bucket \
  --bucket your-terraform-state-bucket \
  --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket your-terraform-state-bucket \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket your-terraform-state-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

### 2. AWS OIDC Provider for GitHub Actions

Create an OIDC identity provider in AWS IAM:

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

### 3. IAM Role for GitHub Actions

Create an IAM role with trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

Attach necessary permissions (EC2, VPC, ECS, EKS, RDS, etc.) to this role.

---

## GitHub Configuration

### 1. Repository Secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret Name | Description | Example |
|------------|-------------|---------|
| `AWS_ROLE_ARN` | IAM role ARN for OIDC | `arn:aws:iam::123456789012:role/github-actions-terraform` |
| `AWS_REGION` | AWS region | `us-east-1` |
| `AWS_ACCOUNT_ID` | AWS account ID | `123456789012` |
| `TF_STATE_BUCKET` | S3 bucket for state | `your-terraform-state-bucket` |
| `TF_STATE_KEY` | State file path | `infra/terraform.tfstate` |
| `TF_STATE_LOCK_TABLE` | DynamoDB table | `terraform-state-lock` |
| `VPC_CIDR` | VPC CIDR block | `192.168.0.0/16` |
| `DB_USERNAME` | RDS master username | `admin` |
| `DB_PASSWORD` | RDS master password | `SecurePassword123!` |
| `DB_NAME` | Initial database name | `cloudnation` |
| `ACM_CERTIFICATE_ARN` | ACM certificate ARN | `arn:aws:acm:us-east-1:123456789012:certificate/xxx` |

### 2. Environment Protection Rules

Configure environments with protection rules:

#### **Development Environment**
- Go to **Settings → Environments → New environment**
- Name: `development`
- No required reviewers (auto-deploy for PRs)

#### **Production Environment**
- Go to **Settings → Environments → New environment**
- Name: `production`
- ✅ Enable **Required reviewers** (add team leads/DevOps engineers)
- ✅ Enable **Wait timer** (optional: 5 minutes)
- ✅ Enable **Deployment branches** → Only `main` branch

#### **Production-Destroy Environment**
- Name: `production-destroy`
- ✅ Enable **Required reviewers** (add multiple senior engineers)
- ✅ Enable **Wait timer** (15 minutes recommended)

---

## Workflow Triggers

### 1. Pull Request (Automatic)
- Triggers on PRs to `main` branch
- Runs: `validate` → `plan`
- Posts plan output as PR comment
- No apply, safe for review

### 2. Push to Main (Manual Approval)
- Triggers on push to `main` branch
- Runs: `validate` → `plan` → **WAIT FOR APPROVAL** → `apply`
- Requires manual approval in GitHub UI

### 3. Manual Workflow Dispatch
- Go to **Actions → Terraform Infrastructure Deployment → Run workflow**
- Choose action: `plan`, `apply`, or `destroy`
- Useful for ad-hoc operations

---

## Usage Workflow

### Making Infrastructure Changes

1. **Create a feature branch**
   ```bash
   git checkout -b feature/add-new-resource
   ```

2. **Make changes in `Infra/` directory**
   ```bash
   # Edit your .tf files
   vim Infra/main.tf
   ```

3. **Format and validate locally** (optional but recommended)
   ```bash
   cd Infra
   terraform fmt -recursive
   terraform init
   terraform validate
   ```

4. **Commit and push**
   ```bash
   git add Infra/
   git commit -m "Add new EKS node group"
   git push origin feature/add-new-resource
   ```

5. **Create Pull Request**
   - GitHub Actions will automatically run `terraform plan`
   - Review the plan output in PR comments
   - Request reviews from team members

6. **Merge to main**
   - After PR approval, merge to `main`
   - Workflow will run `terraform plan` again
   - **Manual approval required** before apply

7. **Approve deployment**
   - Go to **Actions** tab
   - Click on the running workflow
   - Click **Review deployments**
   - Select `production` environment
   - Click **Approve and deploy**

---

## Security Best Practices

✅ **OIDC instead of access keys** - No long-lived credentials stored  
✅ **Least privilege IAM** - Role has only necessary permissions  
✅ **State encryption** - S3 bucket encrypted at rest  
✅ **State locking** - DynamoDB prevents concurrent modifications  
✅ **Manual approval gates** - Human review before production changes  
✅ **Separate destroy environment** - Extra protection for destructive operations  
✅ **Secrets management** - Sensitive values in GitHub Secrets  
✅ **Plan artifacts** - Ensures applied plan matches reviewed plan

---

## Troubleshooting

### Plan fails with "backend initialization required"
- Check S3 bucket and DynamoDB table exist
- Verify IAM role has permissions to access them

### Apply fails with "plan file not found"
- Plan artifact may have expired (5-day retention)
- Re-run the workflow from scratch

### AWS authentication fails
- Verify OIDC provider is configured correctly
- Check IAM role trust policy matches your repository
- Ensure `AWS_ROLE_ARN` secret is correct

### Manual approval not showing
- Verify environment protection rules are configured
- Check you're a designated reviewer for the environment

---

## Advanced: Adding Backend Configuration

If you want to add a backend configuration file instead of using CLI flags:

Create `Infra/backend.tf`:

```hcl
terraform {
  backend "s3" {
    # These will be overridden by -backend-config flags in workflow
    # but provide defaults for local development
    bucket         = "your-terraform-state-bucket"
    key            = "infra/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

---

## Monitoring and Notifications

Consider adding Slack/Teams notifications:

```yaml
- name: Notify on Failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK }}
    payload: |
      {
        "text": "Terraform deployment failed for ${{ github.repository }}"
      }
```

---

## Next Steps

1. Set up all required AWS resources
2. Configure GitHub secrets and environments
3. Test with a small change in a PR
4. Document your specific variable values for your team
5. Consider adding cost estimation with Infracost
6. Set up monitoring for state file changes
