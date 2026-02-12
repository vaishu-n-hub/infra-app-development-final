# AWS Load Balancer Controller - Complete Deployment Explanation

## What is AWS Load Balancer Controller?

The AWS Load Balancer Controller is a Kubernetes controller that manages AWS Elastic Load Balancers (ALBs and NLBs) for your EKS cluster.

**Without it:** You manually create ALBs in AWS console and configure routing.  
**With it:** You create a Kubernetes Ingress resource, and it automatically creates and configures ALBs for you.

---

## The Big Picture: Why We Need Each Step

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: OIDC Provider                                      │
│  Purpose: Allow Kubernetes pods to assume AWS IAM roles    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 2: IAM Policy                                         │
│  Purpose: Define what AWS actions the controller can do    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 3: IAM Role with IRSA                                 │
│  Purpose: Give the controller pod permission to use policy  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 4: Kubernetes Service Account                         │
│  Purpose: Link the IAM role to the controller pod          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 5: Install Controller via Helm                        │
│  Purpose: Deploy the actual controller application         │
└─────────────────────────────────────────────────────────────┘
```

---

## STEP 1: Create OIDC Provider

### What Happens
```bash
OIDC_URL=$(aws eks describe-cluster --name cloud-nation-eks --query "cluster.identity.oidc.issuer" --output text)
OIDC_ID=$(echo $OIDC_URL | cut -d '/' -f 5)

if aws iam list-open-id-connect-providers | grep -q $OIDC_ID; then
  echo "OIDC provider already exists"
else
  eksctl utils associate-iam-oidc-provider --cluster cloud-nation-eks --approve
fi
```

### Why We Do This

**The Problem:**
- Kubernetes pods need to call AWS APIs (create ALBs, modify security groups, etc.)
- Traditional approach: Store AWS access keys in pods (INSECURE!)
- Better approach: Use IAM roles, but pods aren't AWS resources

**The Solution: OIDC (OpenID Connect)**
- OIDC is a trust bridge between Kubernetes and AWS IAM
- It allows Kubernetes service accounts to assume IAM roles
- No credentials stored in pods!

### What Actually Happens

1. **EKS creates an OIDC issuer URL** when cluster is created
   - Example: `https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE`

2. **We register this URL with AWS IAM**
   - Creates an IAM OIDC identity provider
   - AWS now trusts tokens signed by this EKS cluster

3. **Flow when pod needs AWS access:**
   ```
   Pod → Requests token from EKS
   EKS → Signs token with OIDC issuer
   Pod → Sends token to AWS STS
   AWS STS → Validates token against OIDC provider
   AWS STS → Returns temporary credentials
   Pod → Uses credentials to call AWS APIs
   ```

### Real-World Analogy
Think of OIDC like a passport system:
- Your EKS cluster is a country that issues passports (tokens)
- AWS IAM is another country that needs to verify those passports
- OIDC provider is the agreement between countries to trust each other's passports

---


## STEP 2: Download and Create IAM Policy

### What Happens
```bash
# Download the official policy from AWS
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json

# Create the policy in your AWS account
POLICY_NAME="AWSLoadBalancerControllerIAMPolicy-cloud-nation-eks"
aws iam create-policy --policy-name $POLICY_NAME --policy-document file://iam-policy.json
```

### Why We Do This

**The Problem:**
- The controller needs to create/modify AWS resources (ALBs, target groups, security groups, etc.)
- AWS uses IAM policies to define permissions
- We need to tell AWS exactly what the controller is allowed to do

**The Solution:**
- AWS provides an official IAM policy document
- This policy contains the minimum permissions needed
- We create this policy in our AWS account

### What's in the Policy?

The policy allows actions like:
```json
{
  "Effect": "Allow",
  "Action": [
    "elasticloadbalancing:CreateLoadBalancer",
    "elasticloadbalancing:CreateTargetGroup",
    "elasticloadbalancing:CreateListener",
    "elasticloadbalancing:DeleteLoadBalancer",
    "elasticloadbalancing:ModifyLoadBalancerAttributes",
    "ec2:DescribeSubnets",
    "ec2:DescribeSecurityGroups",
    "ec2:CreateSecurityGroup",
    "ec2:AuthorizeSecurityGroupIngress",
    // ... and many more
  ],
  "Resource": "*"
}
```

### Why Download from GitHub?

- AWS updates the policy when new features are added
- Using the official policy ensures compatibility
- Version-specific (v2.13.3) ensures we get the right permissions for that version

### Real-World Analogy
Think of this like a job description:
- The controller is an employee
- The IAM policy is the list of tasks they're allowed to perform
- We're creating this job description in our company (AWS account)

---


## STEP 3: Create IAM Role with IRSA (IAM Roles for Service Accounts)

### What Happens
```bash
POLICY_NAME="AWSLoadBalancerControllerIAMPolicy-cloud-nation-eks"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
POLICY_ARN="arn:aws:iam::${ACCOUNT_ID}:policy/${POLICY_NAME}"

eksctl create iamserviceaccount \
  --cluster=cloud-nation-eks \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=$POLICY_ARN \
  --override-existing-serviceaccounts \
  --approve
```

### Why We Do This

**The Problem:**
- We have an IAM policy (Step 2) that defines permissions
- We have an OIDC provider (Step 1) that enables trust
- But we need to connect them: "Which Kubernetes pod gets which IAM permissions?"

**The Solution: IRSA (IAM Roles for Service Accounts)**
- Creates an IAM role that can be assumed by a specific Kubernetes service account
- Uses OIDC to verify the service account is legitimate
- Provides fine-grained access control (only specific pods get specific permissions)

### What Actually Gets Created

1. **IAM Role** named something like `eksctl-cloud-nation-eks-addon-iamserviceaccount-kube-system-aws-load-balancer-controller`

2. **Trust Policy** on the role:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller",
        "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE:aud": "sts.amazonaws.com"
      }
    }
  }]
}
```

**What this trust policy means:**
- "Allow the OIDC provider (our EKS cluster) to assume this role"
- "BUT ONLY if the request comes from the service account named `aws-load-balancer-controller` in the `kube-system` namespace"
- This is super secure - only that specific pod can use these permissions!

3. **Policy Attachment**
- The IAM policy from Step 2 is attached to this role
- Now the role has both: trust (who can use it) and permissions (what they can do)

4. **Kubernetes Service Account**
- Created in the `kube-system` namespace
- Annotated with the IAM role ARN:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eksctl-cloud-nation-eks-addon-iamserviceaccount...
```

### The Magic: How It Works at Runtime

When the controller pod starts:
```
1. Pod uses service account "aws-load-balancer-controller"
2. AWS SDK in pod sees the annotation with IAM role ARN
3. Pod requests token from Kubernetes API
4. Kubernetes signs token with OIDC issuer
5. Pod sends token to AWS STS with role ARN
6. AWS STS validates:
   - Is token signed by trusted OIDC provider? ✓
   - Does token claim match trust policy conditions? ✓
   - Is service account name correct? ✓
7. AWS STS returns temporary credentials (valid 1 hour)
8. Pod uses credentials to call AWS APIs
9. Process repeats when credentials expire
```

### Real-World Analogy
Think of this like a security badge system:
- The IAM role is a security clearance level
- The service account is a specific employee badge
- IRSA is the system that says "only this badge can access this clearance level"
- The OIDC provider verifies the badge is real and not forged

---


## STEP 4: Add Helm Repository

### What Happens
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

### Why We Do This

**The Problem:**
- The AWS Load Balancer Controller is packaged as a Helm chart
- Helm charts are stored in repositories (like Docker images in registries)
- We need to tell Helm where to find the chart

**The Solution:**
- Add the official AWS EKS Helm repository
- Update the local cache of available charts

### What's a Helm Chart?

A Helm chart is a package containing:
- Kubernetes manifests (Deployment, Service, ConfigMap, etc.)
- Default configuration values
- Templates that can be customized
- Dependencies

Think of it like:
- **Docker image** = Pre-built application
- **Helm chart** = Pre-built Kubernetes application with all its resources

### Why Use Helm?

**Without Helm:**
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f rbac.yaml
# ... 20+ more files
```

**With Helm:**
```bash
helm install my-app eks/aws-load-balancer-controller
```

Helm manages all the resources as a single unit.

### Real-World Analogy
Think of Helm repositories like app stores:
- `https://aws.github.io/eks-charts` is like the AWS App Store
- The ALB controller chart is like an app in that store
- `helm repo add` is like adding the store to your device
- `helm install` is like downloading and installing the app

---


## STEP 5: Install AWS Load Balancer Controller with Helm

### What Happens
```bash
VPC_ID=$(aws eks describe-cluster --name cloud-nation-eks --query "cluster.resourcesVpcConfig.vpcId" --output text)

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=cloud-nation-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID \
  --wait
```

### Why We Do This

**The Problem:**
- We've set up all the permissions and authentication
- But the actual controller application isn't running yet
- We need to deploy the controller pods to our cluster

**The Solution:**
- Use Helm to deploy the controller
- Configure it with our cluster-specific settings
- Link it to the service account we created (which has IAM permissions)

### Breaking Down Each Parameter

#### `helm upgrade --install`
- **upgrade**: If already installed, update it
- **install**: If not installed, install it fresh
- This makes the command idempotent (safe to run multiple times)

#### `aws-load-balancer-controller`
- The release name (what we call this installation)
- Used to manage/upgrade/delete this specific installation

#### `eks/aws-load-balancer-controller`
- `eks/` = The repository we added in Step 4
- `aws-load-balancer-controller` = The chart name

#### `--namespace kube-system`
- Deploy to the `kube-system` namespace
- This is where Kubernetes system components live
- Keeps it separate from application workloads

#### `--set clusterName=cloud-nation-eks`
**Why:** The controller needs to know which EKS cluster it's managing
**What it does:** 
- Tags ALBs with `kubernetes.io/cluster/cloud-nation-eks: owned`
- Helps identify which resources belong to which cluster
- Prevents conflicts in multi-cluster environments

#### `--set serviceAccount.create=false`
**Why:** We already created the service account in Step 3 with IRSA
**What it does:**
- Tells Helm not to create a new service account
- Prevents overwriting our IRSA-configured service account

#### `--set serviceAccount.name=aws-load-balancer-controller`
**Why:** Tell the controller pods to use our IRSA service account
**What it does:**
- Controller pods will use this service account
- This service account has the IAM role annotation
- Pods automatically get AWS permissions via IRSA

#### `--set region=us-east-1`
**Why:** Controller needs to know which AWS region to operate in
**What it does:**
- Creates ALBs in this region
- Queries AWS APIs in this region
- Ensures resources are created in the correct location

#### `--set vpcId=$VPC_ID`
**Why:** Controller needs to know which VPC to create ALBs in
**What it does:**
- Creates ALBs in this VPC
- Validates subnets belong to this VPC
- Ensures network isolation

#### `--wait`
**Why:** Wait for deployment to complete before continuing
**What it does:**
- Waits for pods to be running and healthy
- Fails if deployment doesn't succeed
- Ensures controller is ready before we try to use it

### What Gets Deployed

When you run this command, Helm creates:

1. **Deployment** (2 replicas for high availability)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: aws-load-balancer-controller
  template:
    spec:
      serviceAccountName: aws-load-balancer-controller  # ← Uses our IRSA account
      containers:
      - name: controller
        image: public.ecr.aws/eks/aws-load-balancer-controller:v2.13.3
        args:
        - --cluster-name=cloud-nation-eks
        - --aws-region=us-east-1
        - --aws-vpc-id=vpc-xxxxx
```

2. **Service** (for webhooks)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: aws-load-balancer-webhook-service
  namespace: kube-system
spec:
  ports:
  - port: 443
    targetPort: 9443
```

3. **ClusterRole** (Kubernetes permissions)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: aws-load-balancer-controller
rules:
- apiGroups: [""]
  resources: ["services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "update", "patch"]
# ... more permissions
```

4. **ClusterRoleBinding** (Links ClusterRole to ServiceAccount)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: aws-load-balancer-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: aws-load-balancer-controller
subjects:
- kind: ServiceAccount
  name: aws-load-balancer-controller
  namespace: kube-system
```

5. **ValidatingWebhookConfiguration** (Validates Ingress resources)
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: aws-load-balancer-webhook
webhooks:
- name: vingress.elbv2.k8s.aws
  rules:
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    operations: ["CREATE", "UPDATE"]
```

### Real-World Analogy
Think of this like installing a smart home hub:
- The hub (controller) needs to know your home address (VPC ID)
- It needs to know your region (us-east-1)
- It needs credentials to control devices (IRSA service account)
- Once installed, it watches for commands (Ingress resources)
- When you create a command, it automatically configures devices (creates ALBs)

---


## STEP 6: Verify Deployment

### What Happens
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

### Why We Do This

**The Problem:**
- Helm says it installed successfully, but is it actually working?
- We need to verify the controller pods are running
- We need to check for any errors

**The Solution:**
- Check the deployment status
- Check the pod status
- Look at logs if there are issues

### What to Look For

#### Deployment Status
```bash
$ kubectl get deployment -n kube-system aws-load-balancer-controller

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           2m
```

**Good signs:**
- `READY: 2/2` - Both replicas are running
- `AVAILABLE: 2` - Pods are healthy and ready

**Bad signs:**
- `READY: 0/2` - No pods running (check events)
- `READY: 1/2` - Only one pod running (check logs)

#### Pod Status
```bash
$ kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

NAME                                            READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller-5d8f9c7b9d-abc12   1/1     Running   0          2m
aws-load-balancer-controller-5d8f9c7b9d-xyz34   1/1     Running   0          2m
```

**Good signs:**
- `STATUS: Running` - Pod is running
- `READY: 1/1` - Container is ready
- `RESTARTS: 0` - No crashes

**Bad signs:**
- `STATUS: CrashLoopBackOff` - Pod keeps crashing
- `STATUS: ImagePullBackOff` - Can't pull container image
- `RESTARTS: 5` - Pod has crashed multiple times

### Common Issues and How to Debug

#### Issue 1: Pods Not Starting
```bash
# Check pod events
kubectl describe pod -n kube-system aws-load-balancer-controller-xxxxx

# Common causes:
# - Service account doesn't exist
# - Image pull errors
# - Resource limits too low
```

#### Issue 2: Pods Crashing
```bash
# Check logs
kubectl logs -n kube-system aws-load-balancer-controller-xxxxx

# Common causes:
# - IAM permissions missing
# - IRSA not configured correctly
# - Invalid cluster name or VPC ID
```

#### Issue 3: IRSA Not Working
```bash
# Check service account annotation
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml

# Should see:
# annotations:
#   eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/...

# Check pod environment
kubectl exec -n kube-system aws-load-balancer-controller-xxxxx -- env | grep AWS

# Should see:
# AWS_ROLE_ARN=arn:aws:iam::123456789012:role/...
# AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

### Real-World Analogy
Think of this like checking if your new employee showed up for work:
- Check if they're at their desk (deployment ready)
- Check if they're actually working (pods running)
- Check if they have their access badge (IRSA configured)
- Check if they can access the systems they need (AWS permissions)

---


## How the Controller Works After Installation

Now that the controller is installed, here's what happens when you create an Ingress:

### The Flow

```
1. You create an Ingress resource
   ↓
2. Kubernetes API stores the Ingress
   ↓
3. Controller watches for Ingress changes (via Kubernetes API)
   ↓
4. Controller reads Ingress spec and annotations
   ↓
5. Controller calls AWS APIs to:
   - Create Application Load Balancer
   - Create Target Groups
   - Create Listeners (HTTP/HTTPS)
   - Configure Security Groups
   - Register targets (pod IPs)
   ↓
6. Controller updates Ingress status with ALB DNS name
   ↓
7. Traffic flows: Internet → ALB → Kubernetes Pods
```

### Example: Creating an Ingress

**You create this:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internal
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /customer*
        pathType: ImplementationSpecific
        backend:
          service:
            name: customer-service
            port:
              number: 8080
```

**Controller automatically creates:**

1. **Application Load Balancer**
   - Name: `k8s-default-myingres-abc123`
   - Scheme: Internal (not internet-facing)
   - Subnets: From annotation or auto-discovered
   - Security Group: Auto-created or specified

2. **Target Group**
   - Name: `k8s-default-customer-abc123`
   - Target Type: IP (pod IPs, not node IPs)
   - Port: 8080
   - Health Check: Configured from service

3. **Listener**
   - Port: 443 (HTTPS) or 80 (HTTP)
   - Certificate: From annotation
   - Default Action: Forward to target group

4. **Listener Rules**
   - Path: `/customer*`
   - Action: Forward to customer-service target group

5. **Target Registration**
   - Automatically registers pod IPs as targets
   - Updates when pods scale up/down
   - Removes unhealthy targets

### Controller Responsibilities

The controller continuously:

✅ **Watches** for Ingress/Service changes  
✅ **Creates** ALBs when Ingress is created  
✅ **Updates** ALBs when Ingress is modified  
✅ **Deletes** ALBs when Ingress is deleted  
✅ **Registers** pod IPs as ALB targets  
✅ **Deregisters** pod IPs when pods are deleted  
✅ **Updates** security groups for proper traffic flow  
✅ **Manages** target group health checks  
✅ **Handles** SSL certificates from ACM  
✅ **Configures** ALB attributes (timeouts, stickiness, etc.)

### What You Don't Have to Do Anymore

❌ Manually create ALBs in AWS console  
❌ Manually configure target groups  
❌ Manually register/deregister targets  
❌ Manually update routing rules  
❌ Manually manage security groups  
❌ Manually handle pod scaling  

The controller does all of this automatically!

---


## Security Deep Dive: Why IRSA is Better Than Access Keys

### Old Way (Access Keys) ❌

**How it worked:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
type: Opaque
data:
  access-key-id: QUtJQVlPVVJBQ0NFU1NLRVk=
  secret-access-key: eW91cnNlY3JldGFjY2Vzc2tleQ==
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: controller
        env:
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: access-key-id
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: aws-credentials
              key: secret-access-key
```

**Problems:**
1. **Long-lived credentials** - Keys don't expire automatically
2. **Stored in cluster** - Anyone with cluster access can read them
3. **Hard to rotate** - Need to update secrets and restart pods
4. **Broad permissions** - Usually given admin access "just in case"
5. **No audit trail** - Can't tell which pod made which API call
6. **Credential leakage** - If pod is compromised, keys are exposed

### New Way (IRSA) ✅

**How it works:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ALBController
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: aws-load-balancer-controller
      # No credentials needed!
```

**Benefits:**
1. **Short-lived credentials** - Tokens expire after 1 hour, auto-renewed
2. **No secrets stored** - Credentials generated on-demand
3. **Automatic rotation** - New tokens issued automatically
4. **Least privilege** - Each service account gets only what it needs
5. **Full audit trail** - CloudTrail shows which role made which call
6. **Secure by default** - Even if pod is compromised, credentials expire quickly

### The Security Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  Traditional Access Keys (Insecure)                             │
├─────────────────────────────────────────────────────────────────┤
│  1. Create IAM user                                             │
│  2. Generate access keys (never expire)                         │
│  3. Store keys in Kubernetes secret                             │
│  4. Mount secret in pod                                         │
│  5. Pod uses keys forever                                       │
│  6. If keys leak, attacker has permanent access                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  IRSA (Secure)                                                  │
├─────────────────────────────────────────────────────────────────┤
│  1. Create IAM role with trust policy                           │
│  2. Create service account with role annotation                 │
│  3. Pod starts with service account                             │
│  4. Pod requests token from Kubernetes                          │
│  5. Kubernetes signs token (valid 1 hour)                       │
│  6. Pod exchanges token for AWS credentials via STS             │
│  7. AWS validates token against OIDC provider                   │
│  8. AWS returns temporary credentials (valid 1 hour)            │
│  9. Pod uses credentials                                        │
│  10. After 1 hour, process repeats                              │
│  11. If token leaks, it expires in 1 hour                       │
└─────────────────────────────────────────────────────────────────┘
```

### Real-World Analogy

**Access Keys = House Key**
- Once you have it, you can enter anytime
- If you lose it, someone else can use it
- You have to change all the locks to revoke access
- No way to know who entered when

**IRSA = Hotel Key Card**
- Only works for your room (specific permissions)
- Expires after checkout (1 hour)
- Hotel can see when you used it (CloudTrail)
- If lost, it becomes useless after expiration
- Can be deactivated remotely (revoke role)

---


## Complete Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AWS Account                                        │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  IAM                                                                │    │
│  │                                                                     │    │
│  │  ┌──────────────────────┐      ┌─────────────────────────────┐   │    │
│  │  │ OIDC Provider        │      │ IAM Role                    │   │    │
│  │  │                      │      │ (ALBController)             │   │    │
│  │  │ Issuer: EKS OIDC URL │◄─────│                             │   │    │
│  │  │ Audience: sts.aws... │      │ Trust Policy:               │   │    │
│  │  └──────────────────────┘      │ - Allow OIDC provider       │   │    │
│  │                                 │ - Only for specific SA      │   │    │
│  │                                 │                             │   │    │
│  │                                 │ Attached Policy:            │   │    │
│  │                                 │ - Create/Delete ALB         │   │    │
│  │                                 │ - Modify Target Groups      │   │    │
│  │                                 │ - Manage Security Groups    │   │    │
│  │                                 └─────────────────────────────┘   │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                           │                                  │
│                                           │ AssumeRoleWithWebIdentity        │
│                                           │ (with OIDC token)                │
│                                           ▼                                  │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  AWS STS (Security Token Service)                                  │    │
│  │                                                                     │    │
│  │  1. Receives OIDC token from pod                                   │    │
│  │  2. Validates token signature                                      │    │
│  │  3. Checks trust policy conditions                                 │    │
│  │  4. Returns temporary credentials (1 hour)                         │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                           │                                  │
│                                           │ Temporary Credentials            │
│                                           ▼                                  │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  AWS Services                                                       │    │
│  │                                                                     │    │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐           │    │
│  │  │ ELB/ALB     │  │ EC2          │  │ Target Groups  │           │    │
│  │  │             │  │              │  │                │           │    │
│  │  │ - Create    │  │ - Describe   │  │ - Register     │           │    │
│  │  │ - Delete    │  │ - Modify SG  │  │ - Deregister   │           │    │
│  │  │ - Modify    │  │              │  │                │           │    │
│  │  └─────────────┘  └──────────────┘  └────────────────┘           │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │ API Calls with Temp Credentials
                                    │
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EKS Cluster                                        │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  Namespace: kube-system                                             │    │
│  │                                                                     │    │
│  │  ┌──────────────────────────────────────────────────────────┐     │    │
│  │  │ ServiceAccount: aws-load-balancer-controller             │     │    │
│  │  │                                                           │     │    │
│  │  │ Annotations:                                              │     │    │
│  │  │   eks.amazonaws.com/role-arn: arn:aws:iam::...          │     │    │
│  │  └──────────────────────────────────────────────────────────┘     │    │
│  │                          │                                          │    │
│  │                          │ Uses                                     │    │
│  │                          ▼                                          │    │
│  │  ┌──────────────────────────────────────────────────────────┐     │    │
│  │  │ Deployment: aws-load-balancer-controller                 │     │    │
│  │  │                                                           │     │    │
│  │  │ Replicas: 2                                               │     │    │
│  │  │                                                           │     │    │
│  │  │ ┌─────────────────────────────────────────────────┐     │     │    │
│  │  │ │ Pod 1                                            │     │     │    │
│  │  │ │                                                  │     │     │    │
│  │  │ │ 1. Reads Ingress resources                      │     │     │    │
│  │  │ │ 2. Requests OIDC token from Kubernetes          │     │     │    │
│  │  │ │ 3. Exchanges token for AWS credentials          │     │     │    │
│  │  │ │ 4. Calls AWS APIs to create/update ALBs         │     │     │    │
│  │  │ │ 5. Updates Ingress status                       │     │     │    │
│  │  │ │                                                  │     │     │    │
│  │  │ │ Environment:                                     │     │     │    │
│  │  │ │ - AWS_ROLE_ARN (from SA annotation)             │     │     │    │
│  │  │ │ - AWS_WEB_IDENTITY_TOKEN_FILE (mounted)         │     │     │    │
│  │  │ │ - AWS_REGION                                     │     │     │    │
│  │  │ └─────────────────────────────────────────────────┘     │     │    │
│  │  │                                                           │     │    │
│  │  │ ┌─────────────────────────────────────────────────┐     │     │    │
│  │  │ │ Pod 2 (same as Pod 1)                           │     │     │    │
│  │  │ └─────────────────────────────────────────────────┘     │     │    │
│  │  └──────────────────────────────────────────────────────────┘     │    │
│  │                          │                                          │    │
│  │                          │ Watches                                  │    │
│  │                          ▼                                          │    │
│  │  ┌──────────────────────────────────────────────────────────┐     │    │
│  │  │ Ingress Resources                                         │     │    │
│  │  │                                                           │     │    │
│  │  │ apiVersion: networking.k8s.io/v1                         │     │    │
│  │  │ kind: Ingress                                             │     │    │
│  │  │ metadata:                                                 │     │    │
│  │  │   annotations:                                            │     │    │
│  │  │     alb.ingress.kubernetes.io/scheme: internal           │     │    │
│  │  │ spec:                                                     │     │    │
│  │  │   ingressClassName: alb                                   │     │    │
│  │  └──────────────────────────────────────────────────────────┘     │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---


## Summary: Why Each Step is Critical

### Step 1: OIDC Provider
**Without it:** Pods can't prove their identity to AWS  
**With it:** Kubernetes can issue trusted tokens that AWS accepts  
**Analogy:** Like getting a passport from your country

### Step 2: IAM Policy
**Without it:** Even with identity, pods have no permissions  
**With it:** Defines exactly what AWS actions are allowed  
**Analogy:** Like a job description listing your responsibilities

### Step 3: IAM Role with IRSA
**Without it:** Policy exists but no one can use it  
**With it:** Links the policy to a specific service account  
**Analogy:** Like assigning the job to a specific employee

### Step 4: Helm Repository
**Without it:** Can't find the controller software  
**With it:** Can download and install the controller  
**Analogy:** Like adding an app store to your phone

### Step 5: Install Controller
**Without it:** All permissions set up but no software running  
**With it:** Controller actively manages ALBs  
**Analogy:** Like the employee actually showing up to work

### Step 6: Verify
**Without it:** Assume everything works (dangerous!)  
**With it:** Confirm controller is healthy and working  
**Analogy:** Like checking the employee is at their desk and productive

---

## Common Questions

### Q: Why not just use Terraform to create ALBs?
**A:** You could, but:
- Terraform is manual (you run it)
- Controller is automatic (watches Ingress resources)
- Controller handles pod scaling automatically
- Controller integrates with Kubernetes native resources
- Controller is the Kubernetes-native way

### Q: Can I have multiple controllers in one cluster?
**A:** No, one controller per cluster. It manages all Ingresses with `ingressClassName: alb`.

### Q: What if the controller pods crash?
**A:** 
- You have 2 replicas for high availability
- If both crash, existing ALBs keep working
- New Ingresses won't create ALBs until controller recovers
- Existing ALBs won't update until controller recovers

### Q: How much does this cost?
**A:** 
- Controller pods: Free (run on your EKS nodes)
- ALBs created: ~$16-25/month per ALB
- Data transfer: Standard AWS rates
- OIDC provider: Free
- IAM roles: Free

### Q: Can I use this with ECS?
**A:** No, this is EKS-specific. ECS has its own ALB integration.

### Q: Do I need to install this for every cluster?
**A:** Yes, each EKS cluster needs its own controller installation.

### Q: What happens if I delete the controller?
**A:** 
- Existing ALBs keep working
- You can't create new Ingresses
- You can't update existing Ingresses
- ALBs won't be deleted automatically (manual cleanup needed)

### Q: Can the controller manage existing ALBs?
**A:** No, it only manages ALBs it creates. Don't manually modify controller-created ALBs.

---

## Troubleshooting Checklist

If the controller isn't working, check these in order:

1. ✅ **OIDC Provider exists**
   ```bash
   aws iam list-open-id-connect-providers
   ```

2. ✅ **IAM Policy exists**
   ```bash
   aws iam get-policy --policy-arn arn:aws:iam::ACCOUNT:policy/AWSLoadBalancerControllerIAMPolicy-CLUSTER
   ```

3. ✅ **IAM Role exists with correct trust policy**
   ```bash
   aws iam get-role --role-name eksctl-CLUSTER-addon-iamserviceaccount-kube-system-aws-load-balancer-controller
   ```

4. ✅ **Service Account has role annotation**
   ```bash
   kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
   ```

5. ✅ **Controller pods are running**
   ```bash
   kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
   ```

6. ✅ **Controller logs show no errors**
   ```bash
   kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
   ```

7. ✅ **Ingress has correct ingressClassName**
   ```bash
   kubectl get ingress -A
   # Should show ingressClassName: alb
   ```

8. ✅ **Ingress events show no errors**
   ```bash
   kubectl describe ingress YOUR-INGRESS -n YOUR-NAMESPACE
   ```

---

## Conclusion

The AWS Load Balancer Controller is a powerful tool that bridges Kubernetes and AWS. By understanding each step of the installation process, you can:

- **Troubleshoot** issues when they arise
- **Secure** your cluster with proper IRSA configuration
- **Optimize** your deployment for your specific needs
- **Explain** the architecture to your team

The key insight is that modern cloud-native applications use **identity-based security** (IRSA) instead of **credential-based security** (access keys). This is more secure, more auditable, and more maintainable.

Once installed, the controller works silently in the background, automatically translating your Kubernetes Ingress resources into AWS Application Load Balancers. It's infrastructure as code at its finest - declarative, automated, and reliable.
