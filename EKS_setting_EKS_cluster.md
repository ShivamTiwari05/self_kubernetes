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
```

---------------------****************--------------------****************-------------------***************-------------***************-----------------

****how IAM users/roles can access Kubernetes APIs in Amazon EKS, and how you can configure it in your own cluster****

Here’s a simple and clear breakdown of how IAM users/roles can access Kubernetes APIs in Amazon EKS, and how you can configure it in your own cluster. You can treat this as a conceptual guide when working with EKS.
***What's the Goal Here?***
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

****How to Grant IAM Users Access to Kubernetes****
EKS uses IAM Authenticator, which maps IAM identities to Kubernetes RBAC permissions.

Two methods (authentication modes):
1. aws-auth ConfigMap (Older method)
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

2. Access Entries (Recommended for new clusters)
Introduced in newer EKS platform versions.

You manage access outside the cluster, using:
1. AWS CLI
2. EKS API
3. AWS Console
4. CloudFormation
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
****Mode****	                        ****Description****
CONFIG_MAP	                        Only uses aws-auth ConfigMap (default for old clusters).
API_AND_CONFIG_MAP	                Uses both methods. You manage access via ConfigMap and access entries.
API                        	        Only use access entries. More secure and scalable.

👉 Once you enable access entries (API or API_AND_CONFIG_MAP), you cannot disable them.

****Which Should You Use?****
****if You want...*****                                             ****Use This****
1. Simpler management via AWS CLI/Console	                              Access entries (API or API_AND_CONFIG_MAP)
2. Full control from inside the cluster (legacy clusters)	              aws-auth ConfigMap
3. To migrate old ConfigMap entries	                                    Move to access entries
4. To support hybrid nodes (e.g., EC2 + on-prem)	                      Use API_AND_CONFIG_MAP mode

***Extra Tips***
You can scope access entries by namespace and attach access policies.

If using ConfigMap, always back it up before changes.

For federated users (SSO), use IAM roles and map them via either method.

✅ Summary for Your EKS Setup
1. Use Access Entries if your cluster supports it (modern, scalable).

2. If you're on an old cluster, use aws-auth ConfigMap or enable both (API_AND_CONFIG_MAP).

3. Map IAM roles/users to Kubernetes RBAC groups (e.g., system:masters).

4. Use eksctl or aws eks CLI to manage identity mappings.

5. If you plan to manage access externally (without editing YAMLs inside the cluster), go for Access Entries

****How to Create and Use OIDC for IAM Roles for Service Accounts (IRSA)****
🔹 Step 1: Confirm EKS Cluster Has OIDC Provider
Run:

```
aws eks describe-cluster --name <your-cluster-name> --query "cluster.identity.oidc.issuer" --output text
```
If it returns a URL (like https://oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLE1234), you're good.

If not, add an OIDC provider:

```
eksctl utils associate-iam-oidc-provider \
  --region <region> \
  --cluster <cluster-name> \
  --approve
```
🔹 Step 2: Create an IAM Role for the Pod
The IAM role must trust the OIDC provider and allow a specific Kubernetes namespace and service account to assume it.

You can use eksctl for this:
```
eksctl create iamserviceaccount \
  --name my-sa \
  --namespace default \
  --cluster <cluster-name> \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve \
  --override-existing-serviceaccounts
```
This will:
a. Create a service account my-sa in the default namespace.
b. Create an IAM role that trusts your OIDC provider.
c. Attach the policy to allow S3 read access.

Annotate the service account with the IAM role ARN.

🔹 Step 3: Use the Service Account in a Pod
Here's a simple example:

yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: s3-reader
spec:
  serviceAccountName: my-sa
  containers:
  - name: app
    image: amazonlinux
    command: [ "/bin/sh", "-c", "aws s3 ls" ]
```
This pod will assume the IAM role associated with my-sa and can run AWS CLI calls using that identity.

For External Users via OIDC (e.g., Okta, Google) — [Advanced Use Case]
This involves:
Setting up an OIDC provider (Okta, Google, etc.)
Configuring Kubernetes api-server to trust it (EKS does not yet support direct external OIDC auth for users easily).
You'd usually pair this with Dex or Keycloak and tools like kube-oidc-proxy.

Then, use kubectl with --auth-provider=oidc.

➡️ This is not officially supported natively by EKS out of the box — you need custom setups.

****How to Create OIDC in EKS****
Step	What You Do
1️. Associate your EKS cluster with an IAM OIDC provider using eksctl
2️. Create a Kubernetes service account
3️. Create an IAM role that trusts the OIDC identity and maps to that service account
4️. Deploy workloads using that service account (so pods can assume IAM roles securely)


-------------****************-----------------------***************-----------------***************-----------------**********************---------------



*****Understanding IAM Service Accounts in AWS EKS and Beyond*****
When running applications in AWS EKS (Elastic Kubernetes Service), it’s common for your workloads (pods or containers) to need access to AWS services like S3, DynamoDB, or CloudWatch. However, Kubernetes pods don’t have native AWS identity or permissions. This is where IAM Roles for Service Accounts (IRSA) comes into play — a secure and scalable way to grant AWS permissions to pods running in EKS.

****What is an IAM Service Account?****
In the context of EKS, an IAM service account refers to a Kubernetes service account that is linked to an AWS IAM role. This setup allows the pods that use this service account to assume the IAM role and thereby gain permissions to access AWS services — all without hardcoding credentials or granting over-permissive access to the underlying EC2 nodes.

This is made possible by IRSA, a feature in EKS that uses Kubernetes service accounts in combination with IAM roles and an OIDC identity provider, enabling fine-grained and secure access control.

Why Use IRSA Instead of Other Methods?
Traditionally, developers might be tempted to either bake static AWS credentials into container images (a major security risk), or use the node’s IAM role to grant pods access (which applies the same wide permissions to all pods on a node). Both approaches are risky and not scalable. IRSA solves this problem by allowing each pod to securely assume its own IAM role, scoped to only the AWS resources it needs.

Creating an IAM Service Account Using eksctl
The easiest way to set up IRSA is with the eksctl command-line tool. For example:

bash
Copy
Edit
eksctl create iamserviceaccount \
  --name s3-reader \
  --namespace default \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
This command automates the process of:

Creating a Kubernetes service account (s3-reader in this case),

Creating or attaching an IAM role with the right AWS permissions,

Linking it with the EKS cluster’s OIDC identity provider, and

Annotating the service account so Kubernetes knows which IAM role to use.

Any pod using the s3-reader service account will now be able to securely access S3 using the permissions defined in the IAM policy.

****Manual Setup of IAM Roles for K8s Service Accounts****
***Alternatively, you can manually configure everything:***

1. Create the IAM role with a trust policy that allows the EKS OIDC provider,

2. Annotate the Kubernetes service account with the IAM role’s ARN,
3. And ensure your EKS cluster has OIDC identity provider integration enabled.
   
This gives you full control but requires more setup and knowledge of AWS IAM and Kubernetes internals.

****What is a Kubernetes Service Account?****
A Kubernetes service account is an identity that pods use to interact with the Kubernetes API or external services. It’s not for human users — it’s for internal workloads. Every namespace comes with a default service account, and you can create additional ones as needed. These are used to apply Kubernetes RBAC (Role-Based Access Control) rules or, in the case of IRSA, to assign IAM roles.

To find service accounts in your cluster, you can run:

```
kubectl get serviceaccounts --all-namespaces
```
And to inspect one:

```
kubectl describe serviceaccount s3-reader -n default
```
In a pod spec, you associate the pod with a service account using:
```
yaml
spec:
  serviceAccountName: s3-reader
```
This tells Kubernetes: “run this pod using the s3-reader service account,” and if IRSA is configured, the pod will assume the IAM role associated with it.

****Do Other AWS Services Use Service Accounts?****
The concept of a “service account” is specific to Kubernetes (and platforms like Google Cloud). AWS itself doesn’t have a built-in “service account” object, but it has an equivalent: IAM roles that AWS services assume to interact with other AWS services.

Here are some common examples:

1. EC2 instances use instance roles
2. Lambda functions use execution roles
3. ECS tasks use task roles
4. SageMaker, Glue, and Batch use job execution roles
5. CodeBuild, Step Functions, and EventBridge use service roles

In each case, AWS services are granted permissions to perform actions by assigning them IAM roles — just like Kubernetes pods use service accounts linked to IAM roles through IRSA.

Conclusion
In the cloud-native world, especially with Kubernetes on AWS, managing permissions securely is critical. IRSA (IAM Roles for Service Accounts) offers a powerful, secure, and fine-grained way to manage AWS access for your EKS workloads. Whether you use eksctl for simplicity or configure things manually for more control, understanding how Kubernetes service accounts and AWS IAM work together is key to building secure, scalable cloud applications.

------------------*************--------------------**********************---------------------**********************--------------************--------------------


****IRSA VS RBAC****

IRSA vs RBAC — What’s the Difference?
****Feature****	                                          ****IRSA****	                                                                  ****RBAC****
Layer	                                        AWS IAM (cloud-level)	                                                                Kubernetes (cluster-level)
Purpose	                                      Grant access to AWS services (e.g., S3, DynamoDB) from pods	                          Control access to Kubernetes API (e.g.,                                                                                                                                       who can list pods, create deployments)
Applies To                                   	Pods using a Kubernetes service account	                                               Users, service accounts, or groups
Example	                                      Pod can read from S3 bucket	                                                          User can list pods in a namespace
Defined In	                                  IAM Role + OIDC + IRSA annotation	                                                    Kubernetes Role and RoleBinding or                                                                                                                                             ClusterRole and ClusterRoleBinding

***🔹 Why Do You Need Both?***
You need ***IRSA*** when a pod needs to access AWS resources like: Amazon S3 (download/upload files), Amazon DynamoDB, SQS, SNS, Secrets Manager, etc.

But if you want to control what that pod can do inside the Kubernetes cluster, such as:

Reading ConfigMaps, Writing logs to a volume, Watching other pods, Creating new jobs or services…  you use Kubernetes ***RBAC***.

In short:
IRSA controls what the pod can do in AWS and RBAC controls what the pod (or user) can do in Kubernetes

****Real-World Example****
Let’s say you have a pod that:

1. Needs to fetch configuration from AWS S3
2. Needs to read Secrets from Kubernetes
3. Needs to create Jobs dynamically

You would need: ***IRSA***; so it can access S3 securely. (option 1 and 2)
RBAC RoleBinding so it can access Kubernetes secrets and create jobs (option 2 and 3)

****<u>Imagine You Have a Pod Running in EKS </u>****
That pod (your application) might need to do two different types of things:

1. Access AWS resources
Maybe it needs to read from an S3 bucket; Or talk to DynamoDB or Secrets Manager. This is outside the Kubernetes cluster, in AWS.
For this, you use ****IRSA (IAM Roles for Service Accounts)**** — it gives AWS permissions to the pod.

2. Interact with Kubernetes itself
Maybe it needs to read secrets from Kubernetes; Or create other Kubernetes objects, like jobs or pods; Or get information about services in the cluster. This is inside the Kubernetes cluster. For this, you use RBAC (Role-Based Access Control) — it gives Kubernetes permissions to the pod (or user).

***So Why Do You Need Both?***
***Task***	                                         ***Needs IRSA?***	                                                   ***Needs RBAC?***
Read from S3	                                      ✅ Yes (AWS permission)  	                                             ❌ No
Read a Kubernetes secret	                          ❌ No                                                                  ✅ Yes
List pods in a namespace	                          ❌ No	                                                                 ✅ Yes
Write to DynamoDB	                                  ✅ Yes	                                                                ❌ No
Create a Kubernetes Job	                            ❌ No	                                                                  ✅ Yes
