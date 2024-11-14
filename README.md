# Amazon EKS Cluster Setup and Configuration Guide
**Version:** 1.3  
**Last Updated:** November 14, 2024  
**Author:** DevOps Team

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Cluster Creation](#cluster-creation)
4. [Add-ons Configuration](#addons-configuration)
5. [Cluster Autoscaler Setup](#cluster-autoscaler-setup)
6. [Metrics Server Installation](#metrics-server)
7. [Secrets Management](#secrets-management)
8. [Vertical Pod Autoscaler Setup](#vpa-setup)
9. [Ingress Controller Setup](#ingress-controller)
10. [Network Configuration](#network-configuration)
11. [Troubleshooting](#troubleshooting)

## 1. Introduction <a name="introduction"></a>
This document provides step-by-step instructions for setting up and configuring an Amazon EKS cluster with various components including cluster autoscaling, metrics server, VPA, and choice of ingress controllers.

## 2. Prerequisites <a name="prerequisites"></a>
### 2.1. Required Tools and Permissions
- AWS CLI configured with appropriate permissions
- kubectl installed and configured
- eksctl installed
- Git client
- Helm installed
- Proper IAM permissions to create policies and roles
- SSL certificate files:
  - starinnovapptive2024.pem
  - innovapptive2024.com.key

### 2.2. VPC and Subnet Planning
Before creating the cluster, ensure proper VPC and subnet planning:

1. **CIDR Planning:**
   - Calculate total IP requirements for next year + 25% buffer
   - Consider the following factors:
     - Number of nodes planned
     - Pods per node (default max 110)
     - Services planned
     - Future expansion

2. **Subnet Range Verification:**
   - Check available IPs in each subnet
   - Ensure sufficient IP addresses for:
     - Node Groups
     - Pods (CNI overhead)
     - Internal Services
   - Recommended minimum /24 subnet size for each private subnet

3. **IPv4 Requirements Calculation:**
   ```
   Total IPs needed = (Number of nodes × (1 + max pods per node)) × 1.25 (buffer)
   ```

## 3. Cluster Creation <a name="cluster-creation"></a>
### 3.1. Cluster Configuration File
Create a file named `cluster-config.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: saas-qa-cluster
  region: us-east-1
  version: "1.29"  # One version behind latest stable

vpc:
  id: "vpc-0d5904462fcf31d32"
  subnets:
    public:
      us-east-1a:
        id: "subnet-0d50d1f811d5eef93"
      us-east-1b:
        id: "subnet-078570eb5155995f8"
    private:
      us-east-1a:
        id: "subnet-0e26158e2e7f30b72"
      us-east-1b:
        id: "subnet-0c3356eefc72b3a59"
      us-east-1c:
        id: "subnet-07b551264223821b1"
      us-east-1d:
        id: "subnet-0f66d5667506acdb2"
```

### 3.2. Deploy EKS Cluster
```bash
eksctl create cluster -f cluster-config.yaml
```

### 3.3. Node Group Configuration (Console)
1. Navigate to EKS Console
2. Select your cluster
3. Go to Compute → Node Groups → Add Node Group
4. Configure node group with required specifications
5. Add the following tags during creation:
   ```
   k8s.io/cluster-autoscaler/<cluster-name>: owned
   k8s.io/cluster-autoscaler/enabled: TRUE
   ```

## 4. Add-ons Configuration <a name="addons-configuration"></a>
### 4.1. CNI Add-on
```bash
eksctl create addon --name vpc-cni --cluster saas-qa-cluster --version v1.15.0
```

### 4.2. Kube-proxy Add-on
```bash
eksctl create addon --name kube-proxy --cluster saas-qa-cluster --version v1.29.0-eksbuild.1
```

### 4.3. EBS CSI Add-on
```bash
eksctl create addon --name aws-ebs-csi-driver --cluster saas-qa-cluster --version v1.28.0-eksbuild.1
```

## 5. Cluster Autoscaler Setup <a name="cluster-autoscaler-setup"></a>
### 5.1. Create IAM Policy
```bash
# Create autoscaling policy
aws iam create-policy \
    --policy-name k8s-asg-policy \
    --policy-document file://k8s-asg-policy.json
```

**k8s-asg-policy.json:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": "*"
    }
  ]
}
```

### 5.2. Deploy Cluster Autoscaler
```bash
kubectl apply -f ./cluster-autoscaler-autodiscover.yaml
```

## 6. Metrics Server Installation <a name="metrics-server"></a>
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## 7. Secrets Management <a name="secrets-management"></a>
### 7.1. Create VPA TLS Secret
```bash
# Create TLS secret for VPA
kubectl create secret tls vpa-tls-certs \
    --cert=starinnovapptive2024.pem \
    --key=innovapptive2024.com.key \
    --namespace=kube-system
```

### 7.2. Create Ingress TLS Secret
```bash
# Create TLS secret for Ingress
kubectl create secret tls innovapptive-cert \
    --cert=starinnovapptive2024.pem \
    --key=innovapptive2024.com.key \
    --namespace=kube-system
```

## 8. Vertical Pod Autoscaler Setup <a name="vpa-setup"></a>
### 8.1. Install VPA
```bash
# Clone VPA repository
git clone -b vpa-release-0.9 https://github.com/kubernetes/autoscaler.git

# Navigate to VPA directory
cd autoscaler/vertical-pod-autoscaler

# Remove existing VPA if needed
./hack/vpa-down.sh

# Deploy VPA
./hack/vpa-up.sh
```

### 8.2. Install VPA CRDs and RBAC
```bash
# Install CRDs
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vpa-release-1.0/vertical-pod-autoscaler/deploy/vpa-v1-crd-gen.yaml

# Install RBAC
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/vpa-release-1.0/vertical-pod-autoscaler/deploy/vpa-rbac.yaml

# Verify installation
kubectl get pods -n kube-system | grep vpa
```

## 9. Ingress Controller Setup <a name="ingress-controller"></a>
Choose one of the following ingress controller options:

### 9.1. Option A: AWS Load Balancer Controller
#### 9.1.1. IAM Configuration
```bash
# Download IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

# Create IAM policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# Create service account
eksctl create iamserviceaccount \
    --cluster=<your-cluster-name> \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve
```

#### 9.1.2. Install Cert Manager
```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.13.5/cert-manager.yaml
```

#### 9.1.3. Deploy AWS Load Balancer Controller
```bash
# Download controller manifest
curl -Lo v2_7_2_full.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.7.2/v2_7_2_full.yaml

# Remove ServiceAccount section
sed -i.bak -e '612,620d' ./v2_7_2_full.yaml

# Update cluster name
sed -i.bak -e 's|your-cluster-name|<your-cluster-name>|' ./v2_7_2_full.yaml

# Apply manifest
kubectl apply -f v2_7_2_full.yaml

# Install IngressClass
kubectl apply -f v2_7_2_ingclass.yaml
```

#### 9.1.4. Sample ALB Ingress Configuration
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cbo-preprod-ingress
  namespace: cbopreprod
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: instance
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:1234567890:certificate/5f0b11cf-46db-4461-a0a6-d9cc278c22b4
spec:
  tls:
  - hosts:
    - cbopreprod.innovapptive.com
    secretName: innovapptive-cert
  rules:
  - host: cbopreprod.innovapptive.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### 9.2. Option B: Nginx Ingress Controller
#### 9.2.1. Install Nginx Ingress Controller using Helm
```bash
# Add the Nginx Ingress Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Update Helm repositories
helm repo update

# Install Nginx Ingress Controller
helm install cbo-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace

# Watch the load balancer service status
kubectl get service --namespace ingress-nginx \
    cbo-ingress-ingress-nginx-controller \
    --output wide --watch
```

#### 9.2.2. Create TLS Secret
```bash
# Create TLS secret for your domain
kubectl create secret tls start-innovapptive-ssl \
    --cert=/path/to/starinnovapptive2024.pem \
    --key=/path/to/innovapptive2024.com.key \
    --namespace=cwp
```

#### 9.2.3. Sample Nginx Ingress Configuration
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cwpdeveks-ingress
  namespace: cwp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - cwpeks.innovapptive.com
      secretName: start-innovapptive-ssl
  rules:
    - host: cwpeks.innovapptive.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

## 10. Network Configuration <a name="network-configuration"></a>
### 10.1. VPC Subnet Tagging
Add the following tags to your VPC subnets:

**Public Subnets:**
```
kubernetes.io/role/elb: 1
kubernetes.io/cluster/${your-cluster-name}: owned
```

**Private Subnets:**
```
kubernetes.io/role/internal-elb: 1
kubernetes.io/cluster/${your-cluster-name}: owned
```

> **Note:** Use `shared` instead of `owned` if subnets are used by non-EKS resources.


## 11. Troubleshooting <a name="troubleshooting"></a>
### 11.1. Common Issues
1. **Cluster Autoscaler not working:**
   - Verify IAM policy is attached correctly
   - Check node group tags
   - Review autoscaler logs: `kubectl logs -n kube-system -f deployment/cluster-autoscaler`

2. **Load Balancer Controller issues:**
   - Verify IAM roles and service accounts
   - Check controller logs: `kubectl logs -n kube-system deployment/aws-load-balancer-controller`
   - Ensure subnet tags are correct

3. **VPA issues:**
   - Check VPA pods status: `kubectl get pods -n kube-system | grep vpa`
   - Review VPA logs: `kubectl logs -n kube-system deployment/vpa-recommender`

4. **TLS Secret issues:**
   - Verify secret creation:
     ```bash
     kubectl get secrets -n kube-system | grep tls
     ```
   - Check secret details:
     ```bash
     kubectl describe secret vpa-tls-certs -n kube-system
     kubectl describe secret innovapptive-cert -n kube-system
     ```
   - Ensure certificate files exist and are valid
   - Verify secret is in the correct namespace

### 11.2. Verification Steps
```bash
# Check all pods are running
kubectl get pods -n kube-system

# Verify AWS Load Balancer Controller
kubectl get deployment -n kube-system aws-load-balancer-controller

# Check service account
kubectl get sa -n kube-system aws-load-balancer-controller

# Verify TLS secrets
kubectl get secrets -n kube-system | grep tls

# Check add-ons status
eksctl get addon --cluster saas-qa-cluster
```

---
**End of Documentation**

For any additional support or queries, please contact the DevOps team.
