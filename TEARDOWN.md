# AWS EKS Teardown Guide

This guide documents the cleanup procedure used for this project so the AWS lab environment can be removed without leaving common billable resources behind.

> **Warning**
>
> These commands are destructive. They delete the live Kubernetes application, the ingress load balancer, the Argo CD installation, the EKS worker nodes, and the EKS cluster. The GitHub repository and Docker Hub images are not deleted.

## Environment Used by This Project

```text
AWS region:      us-east-1
EKS cluster:     go-web-app-cluster
Node group:      go-web-app-nodes
Application:     go-web-app
Argo namespace:  argocd
Ingress:         ingress-nginx
```

The cleanup order matters:

```text
Argo CD Application
        ↓
Application Kubernetes resources
        ↓
NGINX LoadBalancer Service
        ↓
Ingress + Argo CD namespaces
        ↓
EKS managed node group
        ↓
EKS control plane
        ↓
Verification for orphaned AWS resources
```

---

# 1. Remove the Argo CD Application

The Argo CD application had automated sync and self-heal enabled. Deleting Kubernetes resources directly while Argo CD is still managing them can cause Argo CD to recreate them.

First, configure a cascading finalizer and delete the Argo CD Application:

```bash
kubectl patch applications.argoproj.io go-web-app -n argocd \
  -p '{"metadata":{"finalizers":["resources-finalizer.argocd.argoproj.io"]}}' \
  --type merge && \
kubectl delete applications.argoproj.io go-web-app -n argocd && \
kubectl get deploy,svc,ingress -n default
```

### What this does

`kubectl patch` adds the Argo CD resource finalizer so Argo CD removes the Kubernetes resources it owns before the Application object disappears.

`kubectl delete applications.argoproj.io` deletes the Argo CD Application.

The final `kubectl get` verifies that the application's Deployment, Service, and Ingress are gone.

It is normal for the built-in Kubernetes Service to remain:

```text
service/kubernetes
```

---

# 2. Capture the Load Balancer Address Before Deletion

Before deleting the ingress controller, capture the AWS load balancer DNS name. This gives you a specific resource to verify later.

```bash
NLB_DNS=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}') && \
echo "Load balancer being removed: $NLB_DNS"
```

Example output:

```text
Load balancer being removed: <AWS-NLB-DNS-NAME>
```

Do not hard-code the temporary AWS load balancer hostname into project files.

---

# 3. Delete the Kubernetes LoadBalancer Service

Delete the ingress controller's `LoadBalancer` Service **before deleting EKS**.

```bash
kubectl delete svc ingress-nginx-controller -n ingress-nginx
```

### Why this is important

A Kubernetes Service of type `LoadBalancer` causes AWS to create an external load balancer. Removing the Kubernetes resource while the cluster is still functional gives the AWS/Kubernetes integration an opportunity to remove that external resource cleanly.

---

# 4. Remove the Platform Namespaces

After the external load balancer Service has been removed:

```bash
kubectl delete namespace ingress-nginx && \
kubectl delete namespace argocd
```

This removes the remaining NGINX Ingress Controller and Argo CD resources from the cluster.

You can combine Steps 3 and 4:

```bash
kubectl delete svc ingress-nginx-controller -n ingress-nginx && \
kubectl delete namespace ingress-nginx && \
kubectl delete namespace argocd
```

---

# 5. Verify `eksctl`

```bash
eksctl version
```

This project was originally provisioned using `eksctl`, so the cluster should also be deleted through `eksctl` rather than trying to remove its CloudFormation resources manually.

---

# 6. Delete the EKS Cluster

```bash
eksctl delete cluster \
  --name go-web-app-cluster \
  --region us-east-1 \
  --wait
```

### What this removes

For the environment created in this project, `eksctl` manages the deletion of the major cluster infrastructure, including:

- EKS control plane
- Managed node group
- EC2 worker nodes
- Node-group CloudFormation stack
- Cluster CloudFormation stack
- Cluster networking resources created by the `eksctl` stack

### Why `--wait` is used

Without `--wait`, the command can return after initiating CloudFormation deletion.

With `--wait`, `eksctl` waits for the deletion process and surfaces failures instead of treating initiation as completion.

A successful teardown should eventually end with a message similar to:

```text
all cluster resources were deleted
```

---

# 7. Verify That EKS Is Gone

Check the region for surviving EKS clusters:

```bash
aws eks list-clusters --region us-east-1
```

Expected result for this lab after teardown:

```json
{
    "clusters": []
}
```

> If you use the same AWS account for other EKS projects, do not expect the entire list to be empty. Instead, verify that `go-web-app-cluster` is absent.

---

# 8. Verify the `eksctl` CloudFormation Stacks Are Gone

```bash
aws cloudformation list-stacks \
  --region us-east-1 \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE DELETE_FAILED \
  --query "StackSummaries[?contains(StackName, 'go-web-app-cluster')].[StackName,StackStatus]" \
  --output table
```

For this project, no surviving cluster stack should be returned.

Pay special attention to:

```text
DELETE_FAILED
```

A stack in `DELETE_FAILED` means cleanup is incomplete and the remaining resources should be investigated.

---

# 9. Check for an Orphaned Network Load Balancer

Use the DNS name captured before deletion:

```bash
aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --query "LoadBalancers[?DNSName=='${NLB_DNS}'].[LoadBalancerName,State.Code]" \
  --output table
```

Expected result:

```text
<no rows>
```

If the old load balancer is still returned after cluster cleanup, investigate it before assuming the AWS bill has stopped.

---

# 10. Check for Surviving EKS EC2 Worker Nodes

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters \
    "Name=tag:aws:eks:cluster-name,Values=go-web-app-cluster" \
    "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[].Instances[].{Instance:InstanceId,State:State.Name,Type:InstanceType}" \
  --output table
```

Expected result:

```text
<no rows>
```

This confirms that no running or stopped EC2 nodes from the cluster remain.

---

# 11. Check for Active NAT Gateways

NAT gateways are independently billable resources, so they are worth checking explicitly.

```bash
aws ec2 describe-nat-gateways \
  --region us-east-1 \
  --filter "Name=state,Values=pending,available" \
  --query "NatGateways[].{NAT:NatGatewayId,State:State,VPC:VpcId}" \
  --output table
```

Expected result for an AWS account used only for this lab:

```text
<no rows>
```

If your account contains other environments, identify each NAT gateway before deleting anything manually.

---

# 12. Check for Unattached EBS Volumes

```bash
aws ec2 describe-volumes \
  --region us-east-1 \
  --filters "Name=status,Values=available" \
  --query "Volumes[].{Volume:VolumeId,SizeGiB:Size,State:State}" \
  --output table
```

An `available` EBS volume is unattached but can still incur storage charges.

Do **not** blindly delete every available EBS volume if the AWS account hosts other projects.

---

# 13. Check for Elastic IP Addresses

```bash
aws ec2 describe-addresses \
  --region us-east-1 \
  --query "Addresses[].{PublicIP:PublicIp,AllocationId:AllocationId,AssociationId:AssociationId}" \
  --output table
```

Review any results carefully.

An unassociated Elastic IP can incur charges. If the account contains unrelated AWS infrastructure, identify ownership before releasing an address.

---

# 14. Final Verification — Lumped Command

After the cluster deletion completes, the following command performs the main read-only verification checks in one pass:

```bash
echo "===== EKS CLUSTERS =====" && \
aws eks list-clusters --region us-east-1 && \
echo "===== EKS CLOUDFORMATION STACKS =====" && \
aws cloudformation list-stacks \
  --region us-east-1 \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE DELETE_FAILED \
  --query "StackSummaries[?contains(StackName, 'go-web-app-cluster')].[StackName,StackStatus]" \
  --output table && \
echo "===== EKS EC2 NODES =====" && \
aws ec2 describe-instances \
  --region us-east-1 \
  --filters \
    "Name=tag:aws:eks:cluster-name,Values=go-web-app-cluster" \
    "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "Reservations[].Instances[].{Instance:InstanceId,State:State.Name,Type:InstanceType}" \
  --output table && \
echo "===== ACTIVE NAT GATEWAYS =====" && \
aws ec2 describe-nat-gateways \
  --region us-east-1 \
  --filter "Name=state,Values=pending,available" \
  --query "NatGateways[].{NAT:NatGatewayId,State:State,VPC:VpcId}" \
  --output table && \
echo "===== UNATTACHED EBS VOLUMES =====" && \
aws ec2 describe-volumes \
  --region us-east-1 \
  --filters "Name=status,Values=available" \
  --query "Volumes[].{Volume:VolumeId,SizeGiB:Size,State:State}" \
  --output table && \
echo "===== ELASTIC IPS =====" && \
aws ec2 describe-addresses \
  --region us-east-1 \
  --query "Addresses[].{PublicIP:PublicIp,AllocationId:AllocationId,AssociationId:AssociationId}" \
  --output table
```

If you captured the old load balancer DNS name in `NLB_DNS`, add this check separately:

```bash
aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --query "LoadBalancers[?DNSName=='${NLB_DNS}'].[LoadBalancerName,State.Code]" \
  --output table
```

---

# 15. What Remains After Teardown

Deleting the AWS environment does **not** remove the project itself.

The following remain available for rebuilding:

```text
GitHub repository
├── Go application
├── Dockerfile
├── Kubernetes manifests
├── Helm chart
├── GitHub Actions workflow
├── README
└── teardown documentation

Docker Hub
└── previously published application images
```

The destroyed resources are the live infrastructure:

```text
EKS control plane
EC2 worker nodes
Kubernetes Pods
Kubernetes Services
Ingress resources
NGINX Ingress Controller
Argo CD installation
AWS load balancer
cluster networking created for the lab
```

---

# 16. Rebuilding the Environment

Project 1 created the infrastructure primarily with `eksctl` and then installed the platform components separately.

The next iteration of this project is intended to rebuild this environment with **Terraform**, so infrastructure creation and destruction become declarative and repeatable:

```text
terraform init
      ↓
terraform plan
      ↓
terraform apply
      ↓
EKS + platform infrastructure
      ↓
GitOps application deployment
      ↓
terraform destroy
```

This teardown guide therefore serves as the final lifecycle chapter for the original `eksctl` implementation and the baseline for the Terraform rebuild.
