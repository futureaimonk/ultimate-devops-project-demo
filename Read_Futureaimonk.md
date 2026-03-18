# AWS EKS Production Deployment Guide
## Project: futureaimonk.me - OpenTelemetry Demo (20 Microservices)

Contents
Architecture Overview - Request flow diagram
Prerequisites - Required tools (AWS CLI, kubectl, eksctl, Helm)
DNS Configuration Options - Squarespace vs Cloudflare vs Route 53 comparison
EKS Cluster Creation - eksctl and Terraform options
AWS Load Balancer Controller Setup - Complete IRSA setup with IAM policies
Application Deployment - Ingress configuration for ALB
HTTPS/SSL Configuration - ACM certificate setup
DNS Setup - Squarespace, GoDaddy, and Cloudflare instructions
Verification Steps - Commands to validate deployment
Cleanup/Deletion Steps - Ordered deletion to avoid orphaned resources
Cost Estimation - ~$180/month breakdown
Troubleshooting - Common issues and solutions
Quick Reference Commands - Handy kubectl commands
The file is ready for your production reference!
---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [DNS Configuration Options](#dns-configuration-options)
4. [EKS Cluster Creation](#eks-cluster-creation)
5. [AWS Load Balancer Controller Setup](#aws-load-balancer-controller-setup)
6. [Application Deployment](#application-deployment)
7. [HTTPS/SSL Configuration](#httpsssl-configuration)
8. [DNS Setup (Squarespace/GoDaddy/Cloudflare)](#dns-setup)
9. [Verification Steps](#verification-steps)
10. [Cleanup/Deletion Steps](#cleanupdeletion-steps)
11. [Cost Estimation](#cost-estimation)
12. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────┐
│   User      │────▶│  DNS Provider   │────▶│    AWS ALB      │────▶│  EKS Pods   │
│  Browser    │     │ (Squarespace/   │     │ (Load Balancer) │     │ (20 micro-  │
│             │     │  Cloudflare)    │     │                 │     │  services)  │
└─────────────┘     └─────────────────┘     └─────────────────┘     └─────────────┘
```

### Request Flow
```
1. User types: www.futureaimonk.me
         ↓
2. DNS Resolution: Squarespace/Cloudflare → Returns ALB IP
         ↓
3. User connects to AWS ALB
         ↓
4. ALB routes to EKS pods based on Ingress rules
         ↓
5. Response flows back to user
```

---

## Prerequisites

### Local Tools Required
```bash
# AWS CLI
brew install awscli
aws configure  # Set region: us-east-1 (or your preferred region)

# kubectl
brew install kubectl

# eksctl
brew install eksctl

# Helm
brew install helm
```

### AWS Account Requirements
- IAM user with Admin access or specific EKS/EC2/IAM permissions
- VPC limits (default is sufficient)

---

## DNS Configuration Options

| Approach | Cost | Best For |
|----------|------|----------|
| **Squarespace/GoDaddy DNS → ALB** | Free | Dev/POC projects |
| **Cloudflare DNS → ALB** | Free | Production (adds CDN, DDoS protection) |
| **Route 53 → ALB** | $0.50/month | AWS-native features (health checks, failover) |

**Recommendation:** Use Cloudflare (free) for production or Squarespace/GoDaddy for dev/POC.

---

## EKS Cluster Creation

### Option 1: Using eksctl (Recommended)
```bash
# Create EKS cluster with managed node group
eksctl create cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --nodegroup-name general \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed

# Verify cluster
kubectl get nodes
```

### Option 2: Using Terraform
See `terraform/` directory for IaC approach.

### Update kubeconfig
```bash
aws eks update-kubeconfig --name my-eks-cluster --region us-east-1
```

---

## AWS Load Balancer Controller Setup

### Step 1: Download IAM Policy
```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

### Step 2: Create IAM Policy
```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### Step 3: Get OIDC Provider
```bash
# Get OIDC ID
oidc_id=$(aws eks describe-cluster --name my-eks-cluster --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id

# Associate OIDC provider (if not already done)
eksctl utils associate-iam-oidc-provider --cluster my-eks-cluster --approve
```

### Step 4: Create Trust Policy
Create `trust-policy.json`:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:aud": "sts.amazonaws.com",
                    "oidc.eks.<REGION>.amazonaws.com/id/<OIDC_ID>:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
                }
            }
        }
    ]
}
```

### Step 5: Create IAM Role
```bash
aws iam create-role \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

### Step 6: Install ALB Controller via Helm
```bash
# Add Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Get VPC ID
VPC_ID=$(aws eks describe-cluster --name my-eks-cluster --query "cluster.resourcesVpcConfig.vpcId" --output text)

# Install controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKSLoadBalancerControllerRole \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID

# Verify installation
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

---

## Application Deployment

### Deploy All Microservices
```bash
cd kubernetes/
kubectl apply -f .
```

### Ingress Configuration (kubernetes/frontendproxy/ingress.yaml)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-proxy
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /
spec:
  ingressClassName: alb
  rules:
    - host: futureaimonk.me
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: opentelemetry-demo-frontendproxy
                port:
                  number: 8080
    - host: www.futureaimonk.me
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: opentelemetry-demo-frontendproxy
                port:
                  number: 8080
```

### Apply Ingress
```bash
kubectl apply -f frontendproxy/ingress.yaml

# Get ALB DNS name
kubectl get ingress frontend-proxy
# Note the ADDRESS column - this is your ALB DNS
```

---

## HTTPS/SSL Configuration

### Step 1: Request ACM Certificate
```bash
aws acm request-certificate \
  --domain-name futureaimonk.me \
  --subject-alternative-names "*.futureaimonk.me" \
  --validation-method DNS \
  --region us-east-1
```

### Step 2: Validate Certificate
Add the CNAME record provided by ACM to your DNS (Route 53 or external DNS).

### Step 3: Update Ingress for HTTPS
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-proxy
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:<ACCOUNT_ID>:certificate/<CERT_ID>
spec:
  ingressClassName: alb
  rules:
    - host: futureaimonk.me
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: opentelemetry-demo-frontendproxy
                port:
                  number: 8080
    - host: www.futureaimonk.me
      http:
        paths:
          - path: "/"
            pathType: Prefix
            backend:
              service:
                name: opentelemetry-demo-frontendproxy
                port:
                  number: 8080
```

---

## DNS Setup

### Option A: Squarespace DNS (Used in this project)
1. Login to Squarespace → Settings → Domains → Select domain
2. Go to DNS Settings → Add Record
3. Add CNAME record:
   | Host | Type | Value |
   |------|------|-------|
   | `www` | CNAME | `<ALB-DNS>.us-east-1.elb.amazonaws.com` |

### Option B: GoDaddy DNS
1. Login to GoDaddy → My Products → DNS
2. Add CNAME record pointing to ALB DNS

### Option C: Cloudflare (Recommended for Production)
1. Create Cloudflare account → Add site
2. Update nameservers in Squarespace/GoDaddy to Cloudflare's NS
3. In Cloudflare DNS, add CNAME record (Proxied)
4. Enable SSL/TLS → Full (Strict)

---

## Verification Steps

```bash
# Check DNS resolution
nslookup www.futureaimonk.me

# Test HTTP response
curl -s -o /dev/null -w "%{http_code}" http://www.futureaimonk.me

# Check pods are running
kubectl get pods

# Check ingress status
kubectl get ingress frontend-proxy
kubectl describe ingress frontend-proxy

# Check ALB controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50
```

---

## Cleanup/Deletion Steps

### ⚠️ IMPORTANT: Follow this order to avoid orphaned resources

### Step 1: Delete Kubernetes Ingress (removes ALB)
```bash
kubectl delete ingress frontend-proxy
kubectl delete -f kubernetes/
```

### Step 2: Delete EKS Node Group
```bash
# List node groups
aws eks list-nodegroups --cluster-name my-eks-cluster --region us-east-1

# Delete node group
aws eks delete-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name general \
  --region us-east-1

# Wait for deletion (5-10 minutes)
aws eks describe-nodegroup --cluster-name my-eks-cluster --nodegroup-name general --region us-east-1 --query 'nodegroup.status'
```

### Step 3: Delete EKS Cluster
```bash
# Only after node group is deleted
aws eks delete-cluster --name my-eks-cluster --region us-east-1

# Or use eksctl
eksctl delete cluster --name my-eks-cluster --region us-east-1
```

### Step 4: Delete ACM Certificate
```bash
aws acm delete-certificate --certificate-arn arn:aws:acm:us-east-1:<ACCOUNT_ID>:certificate/<CERT_ID> --region us-east-1
```

### Step 5: Delete Route 53 Hosted Zone (if created)
```bash
# List hosted zones
aws route53 list-hosted-zones

# Delete records first, then hosted zone
aws route53 delete-hosted-zone --id <HOSTED_ZONE_ID>
```

### Step 6: Delete IAM Resources
```bash
# Detach policies from role
aws iam detach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy

# Delete role
aws iam delete-role --role-name AmazonEKSLoadBalancerControllerRole

# Delete policy
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy
```

### Step 7: Verify All Resources Deleted
```bash
# Check EKS clusters
aws eks list-clusters --region us-east-1

# Check VPCs
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=*eks*" --region us-east-1

# Check Load Balancers
aws elbv2 describe-load-balancers --region us-east-1

# Check ACM certificates
aws acm list-certificates --region us-east-1

# Check Route 53
aws route53 list-hosted-zones
```

---

## Cost Estimation

| Resource | Approx. Cost/Month |
|----------|-------------------|
| EKS Control Plane | $72 |
| 2x t3.medium nodes | ~$60 |
| NAT Gateway | ~$32 |
| ALB | ~$16 + data transfer |
| Route 53 Hosted Zone | $0.50 (optional) |
| ACM Certificate | Free |
| **Total (minimal)** | **~$180/month** |

### Cost Saving Tips
- Use Spot instances for non-production
- Use smaller instance types for dev
- Delete resources when not in use
- Use Squarespace/Cloudflare DNS instead of Route 53

---

## Troubleshooting

### Issue: ALB not being created
```bash
# Check ALB controller logs
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=100

# Common causes:
# - Missing IAM permissions
# - IRSA not configured correctly
# - Subnets not tagged properly
```

### Issue: SSL Certificate errors (Corporate Proxy)
```bash
# Add --no-verify-ssl flag to AWS CLI commands
aws eks describe-cluster --name my-eks-cluster --no-verify-ssl

# For kubectl, update kubeconfig
kubectl config set-cluster <cluster-name> --insecure-skip-tls-verify=true
```

### Issue: Ingress shows no ADDRESS
```bash
# Check ingress events
kubectl describe ingress frontend-proxy

# Verify ALB controller is running
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

### Issue: 404 when accessing domain
- Ensure Ingress `host` matches the domain you're accessing
- Check if both `futureaimonk.me` and `www.futureaimonk.me` are in Ingress rules

---

## Quick Reference Commands

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes

# Pods status
kubectl get pods -A
kubectl get pods -o wide

# Services
kubectl get svc

# Ingress
kubectl get ingress
kubectl describe ingress frontend-proxy

# Logs
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # Follow logs

# ALB Controller
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50
```

---

## Project Files Structure

```
ultimate-devops-project-demo/
├── kubernetes/
│   ├── frontendproxy/
│   │   ├── deploy.yaml
│   │   ├── svc.yaml
│   │   └── ingress.yaml      # ALB Ingress configuration
│   ├── frontend/
│   ├── cart/
│   ├── checkout/
│   ├── ... (other microservices)
│   └── complete-deploy.yaml
├── terraform/                 # IaC for EKS (optional)
│   ├── providers.tf
│   ├── variables.tf
│   ├── vpc.tf
│   ├── eks.tf
│   ├── alb-controller.tf
│   └── outputs.tf
└── Read_Futureaimonk.md      # This documentation
```

---

## Author
**Project:** futureaimonk.me  
**Date:** February 2026  
**Stack:** AWS EKS, ALB, Kubernetes, OpenTelemetry Demo (20 Microservices)

---

*This documentation is for production reference. Always test in a non-production environment first.*
