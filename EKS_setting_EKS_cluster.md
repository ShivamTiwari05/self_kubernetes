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

****while creating the cluster in EKS, these are the things which are considered:****  

IRSA : which allow pods to access resources  

additional access points = which allows other IAM roles/Users to access your cluster.

