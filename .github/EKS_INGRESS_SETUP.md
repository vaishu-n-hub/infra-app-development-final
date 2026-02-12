# EKS Ingress and ALB Controller Setup Guide

## Overview
This workflow deploys the AWS Load Balancer Controller and Kubernetes Ingress resources to your EKS cluster using GitHub Actions with OIDC authentication.

## Architecture

```
Internet → Frontend ALB (ECS) → ECS Services
                                    ↓
                            Backend ALB (EKS Ingress)
                                    ↓
                        EKS Services (customer, driver)
                                    ↓
                                RDS MySQL
```

## Prerequisites

### 1. EKS Cluster Must Be Running
Ensure your Terraform infrastructure has been deployed first:
```bash
# Run the terraform-deploy workflow first
```

### 2. Install eksctl (for OIDC setup)
The workflow uses `eksctl` for OIDC provider association. It's pre-installed on GitHub Actions runners.

### 3. Microservices Must Be Deployed
Before deploying ingress, ensure your backend services are running:
- `customer-service` (port 8080)
- `driver-service` (port 8080)

---

## GitHub Secrets Required

Add these secrets in **Settings → Secrets and variables → Actions**:

| Secret Name | Description | Example |
|------------|-------------|---------|
| `AWS_ROLE_ARN` | IAM role ARN for OIDC | `arn:aws:iam::123456789012:role/github-actions-eks` |
| `AWS_REGION` | AWS region | `us-east-1` |
| `EKS_CLUSTER_NAME` | EKS cluster name | `cloud-nation-eks` |
| `ACM_CERTIFICATE_ARN` | ACM certificate for backend ALB | `arn:aws:acm:us-east-1:123456789012:certificate/xxx` |

---

## Workflow Jobs

### 1. Deploy ALB Controller
Installs the AWS Load Balancer Controller in your EKS cluster.

**What it does:**
- Creates OIDC provider for EKS (if not exists)
- Creates IAM policy for ALB controller
- Creates IAM role with IRSA (IAM Roles for Service Accounts)
- Installs ALB controller via Helm
- Verifies deployment

**When it runs:**
- Manual trigger with `deploy-alb-controller` or `deploy-all` action
- Only needs to run once per cluster

### 2. Deploy Ingress
Deploys the Kubernetes Ingress resource that creates an internal ALB.

**What it does:**
- Fetches app-tier subnet IDs from AWS
- Fetches backend ALB security group ID
- Creates Helm values file with dynamic configuration
- Deploys ingress using Helm
- Verifies ALB creation
- Outputs ALB DNS name

**When it runs:**
- After ALB controller deployment
- On push to main (if ingress files changed)
- Manual trigger with `deploy-ingress` or `deploy-all` action

### 3. Validate Ingress (PR Only)
Validates Helm chart syntax on pull requests.

**What it does:**
- Lints Helm chart
- Validates template rendering
- Posts validation results as PR comment

### 4. Destroy Ingress
Removes the ingress resource and associated ALB.

**When it runs:**
- Manual trigger with `destroy-ingress` action
- Requires approval in `production-destroy` environment

---

## Usage

### First Time Setup (Complete Deployment)

1. **Deploy Infrastructure First**
   ```bash
   # Run terraform-deploy workflow
   # Wait for EKS cluster to be ready
   ```

2. **Deploy Microservices**
   ```bash
   # Deploy customer-service and driver-service to EKS
   # Ensure they're running on port 8080
   ```

3. **Deploy ALB Controller and Ingress**
   - Go to **Actions → EKS Ingress and ALB Controller Deployment**
   - Click **Run workflow**
   - Select `deploy-all`
   - Click **Run workflow**
   - Approve in production environment

4. **Verify Deployment**
   ```bash
   # Check ALB controller
   kubectl get deployment -n kube-system aws-load-balancer-controller
   
   # Check ingress
   kubectl get ingress -n default
   
   # Get ALB DNS
   kubectl get ingress ingress-hpm -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
   ```

### Update Ingress Configuration

1. **Modify ingress files**
   ```bash
   git checkout -b feature/update-ingress
   # Edit App/Harness/Backend/ingress/values.yaml or templates/ingress.yaml
   git commit -am "Update ingress routes"
   git push
   ```

2. **Create PR**
   - Workflow validates Helm chart automatically
   - Review validation results

3. **Merge to main**
   - Workflow deploys updated ingress
   - Requires manual approval

### Add New Route to Ingress

Edit `App/Harness/Backend/ingress/values.yaml`:

```yaml
routes:
  - path: /customer*
    serviceName: customer-service
    servicePort: 8080
    
  - path: /driver*
    serviceName: driver-service
    servicePort: 8080
  
  # Add new route
  - path: /booking*
    serviceName: booking-service
    servicePort: 8080
```

Commit and push to trigger deployment.

---

## Ingress Configuration Details

### Annotations Explained

```yaml
annotations:
  # Internal ALB (not internet-facing)
  alb.ingress.kubernetes.io/scheme: internal
  
  # Deploy in app-tier subnets (private)
  alb.ingress.kubernetes.io/subnets: subnet-xxx,subnet-yyy,subnet-zzz
  
  # Target type IP (for Fargate/EKS pods)
  alb.ingress.kubernetes.io/target-type: ip
  
  # Health check endpoint
  alb.ingress.kubernetes.io/healthcheck-path: /actuator/health
  
  # HTTPS listener on port 443
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
  
  # SSL certificate
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
  
  # Security group (allows traffic from ECS)
  alb.ingress.kubernetes.io/security-groups: sg-xxx
```

### Subnet Selection

The workflow automatically selects app-tier subnets:
- `app-tier-subnet-1` (AZ-1)
- `app-tier-subnet-2` (AZ-2)
- `app-tier-subnet-3` (AZ-3)

These are private subnets where your EKS pods run.

### Security Group

The backend ALB security group (`backend-alb-sg`) allows:
- **Ingress:** HTTPS (443) from ECS security group
- **Egress:** All traffic

This ensures only your ECS frontend can reach the backend services.

---

## Troubleshooting

### ALB Controller Not Installing

**Check OIDC provider:**
```bash
aws iam list-open-id-connect-providers
```

**Check service account:**
```bash
kubectl get sa aws-load-balancer-controller -n kube-system
kubectl describe sa aws-load-balancer-controller -n kube-system
```

**Check controller logs:**
```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

### Ingress Not Creating ALB

**Check ingress status:**
```bash
kubectl describe ingress ingress-hpm -n default
```

**Common issues:**
- Services don't exist (deploy microservices first)
- Subnets not found (check subnet tags)
- Security group not found (check Terraform outputs)
- Certificate ARN invalid (verify in ACM)

**Check controller logs:**
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

### ALB Created But Not Routing

**Check target health:**
```bash
# Get ALB ARN from AWS console
aws elbv2 describe-target-health --target-group-arn <target-group-arn>
```

**Check service endpoints:**
```bash
kubectl get endpoints customer-service -n default
kubectl get endpoints driver-service -n default
```

**Verify pods are running:**
```bash
kubectl get pods -n default
```

### Permission Errors

**Verify IAM role has correct trust policy:**
```bash
aws iam get-role --role-name github-actions-eks
```

**Check IRSA annotation:**
```bash
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
```

---

## Manual Deployment (Local Testing)

If you want to test locally:

```bash
# 1. Configure AWS credentials
export AWS_PROFILE=your-profile

# 2. Update kubeconfig
aws eks update-kubeconfig --name cloud-nation-eks --region us-east-1

# 3. Get subnet IDs
SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=app-tier-subnet-*" \
  --query "Subnets[*].SubnetId" \
  --output text)

# 4. Get security group
BACKEND_ALB_SG=$(aws ec2 describe-security-groups \
  --filters "Name=tag:Name,Values=backend-alb-sg" \
  --query "SecurityGroups[0].GroupId" \
  --output text)

# 5. Deploy with Helm
helm upgrade --install backend-ingress \
  App/Harness/Backend/ingress \
  --set ingress.certificateArn=<your-cert-arn> \
  --set ingress.securitygroup=$BACKEND_ALB_SG \
  --set ingress.subnets.subnetA=<subnet-1> \
  --set ingress.subnets.subnetB=<subnet-2> \
  --set ingress.subnets.subnetC=<subnet-3> \
  --namespace default
```

---

## Cleanup

### Remove Ingress Only
```bash
# Via workflow
Actions → Run workflow → Select "destroy-ingress"

# Or manually
helm uninstall backend-ingress --namespace default
```

### Remove ALB Controller
```bash
# Uninstall Helm release
helm uninstall aws-load-balancer-controller -n kube-system

# Delete service account
kubectl delete sa aws-load-balancer-controller -n kube-system

# Delete IAM resources (manual)
aws iam detach-role-policy --role-name EKS-ALB-Controller-cloud-nation-eks --policy-arn <policy-arn>
aws iam delete-role --role-name EKS-ALB-Controller-cloud-nation-eks
aws iam delete-policy --policy-arn <policy-arn>
```

---

## Best Practices

1. **Deploy ALB controller once** - It manages all ingresses in the cluster
2. **Use internal ALB** - Backend services shouldn't be internet-facing
3. **Health checks** - Ensure services have `/actuator/health` or similar endpoint
4. **Certificate management** - Use ACM for automatic renewal
5. **Security groups** - Restrict access to only necessary sources
6. **Monitoring** - Check ALB metrics in CloudWatch
7. **Cost optimization** - Internal ALB is cheaper than internet-facing

---

## Next Steps

1. Set up monitoring for ALB (CloudWatch, Prometheus)
2. Configure ALB access logs to S3
3. Add WAF rules for additional security
4. Set up auto-scaling for backend services
5. Implement circuit breakers and retries
6. Add observability (tracing, logging)
