***Prerequisites***  

Download and Install AWS Cli - Please Refer this link.
Setup and configure AWS CLI using the aws configure command.
Install and configure eksctl using the steps mentioned here.
Install and configure kubectl as mentioned here.


```
eksctl create cluster --name=observability \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup
```

```
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster observability \
    --approve
```
```
eksctl create nodegroup --cluster=observability \
                        --region=us-east-1 \
                        --name=observability-ng-private \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=3 \
                        --node-volume-size=20 \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking

# Update ./kube/config file
aws eks update-kubeconfig --name observability



****Understanduing of how IAM users/roles can access Kubernetes APIs in Amazon EKS, and how you can configure it in your own cluster****

Here’s a simple and clear breakdown of how IAM users/roles can access Kubernetes APIs in Amazon EKS, and how you can configure it in your own cluster. You can treat this as a conceptual guide when working with EKS.
🧠 What's the Goal Here?
You want IAM users or roles (e.g., your AWS login or CI/CD role) to be able to:

1. Use kubectl to interact with Kubernetes objects in your EKS cluster ✅
2. Manage the EKS cluster itself (via AWS CLI, Console, or SDKs) ✅

🔐 Types of Identities That Can Access the Cluster
1. IAM Users and Roles (Most common):
These identities authenticate through AWS IAM (via access keys, SSO, federation, etc.).
You can assign:
Kubernetes permissions (e.g., read pods, deploy apps)
AWS/EKS permissions (e.g., create cluster, upgrade EKS)
✅ Best for: Admins, CI/CD tools, developers using AWS.

2. OIDC Provider Users (Less common)
You set up your own OIDC provider (like Okta, Auth0, Google, etc.).
Can only assign Kubernetes permissions.
Cannot use AWS CLI/Console to manage EKS or cluster-level settings.
✅ Best for: External user auth or enterprise SSO solutions.

🔧 How to Grant IAM Users Access to Kubernetes
EKS uses IAM Authenticator, which maps IAM identities to Kubernetes RBAC permissions.

Two methods (authentication modes):
🔁 1. aws-auth ConfigMap (Older method)
This ConfigMap lives inside the cluster.

The person who creates the cluster is the only one with access by default.

That person must manually add others via:

```
eksctl create iamidentitymapping \
  --cluster my-cluster \
  --arn arn:aws:iam::111122223333:user/my-user \
  --username my-user \
  --group system:masters
  ```
Users added this way get mapped to Kubernetes users and groups (for RBAC).
You must manage this ConfigMap manually.

🆕 2. Access Entries (Recommended for new clusters)
Introduced in newer EKS platform versions.

You manage access outside the cluster, using:

AWS CLI

EKS API

AWS Console

CloudFormation

✅ No need to SSH into the cluster to modify ConfigMap.

Example CLI to create access entry:

```
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::111122223333:user/my-user \
  --type STANDARD \
  --kubernetes-group system:masters
  ```
⚙️ Authentication Modes Explained
Mode	                        Description
CONFIG_MAP	                Only uses aws-auth ConfigMap (default for old clusters).
API_AND_CONFIG_MAP	        Uses both methods. You manage access via ConfigMap and access entries.
API                        	Only use access entries. More secure and scalable.

👉 Once you enable access entries (API or API_AND_CONFIG_MAP), you cannot disable them.

🗺️ Which Should You Use?
You want...	                                                        Use This
Simpler management via AWS CLI/Console	                           ✅ Access entries (API or API_AND_CONFIG_MAP)
Full control from inside the cluster (legacy clusters)	            aws-auth ConfigMap
To migrate old ConfigMap entries	                                  Move to access entries
To support hybrid nodes (e.g., EC2 + on-prem)	                      Use API_AND_CONFIG_MAP mode

💡 Extra Tips
You can scope access entries by namespace and attach access policies.

If using ConfigMap, always back it up before changes.

For federated users (SSO), use IAM roles and map them via either method.

✅ Summary for Your EKS Setup
1. Use Access Entries if your cluster supports it (modern, scalable).

2. If you're on an old cluster, use aws-auth ConfigMap or enable both (API_AND_CONFIG_MAP).

3. Map IAM roles/users to Kubernetes RBAC groups (e.g., system:masters).

4. Use eksctl or aws eks CLI to manage identity mappings.

5. If you plan to manage access externally (without editing YAMLs inside the cluster), go for Access Entries

```
