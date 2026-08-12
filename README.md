# Kubernetes End‑to‑End on AWS EKS 

****Demo summary**  
This repo documents an end‑to‑end demo that provisions an Amazon EKS cluster, configures IAM/OIDC for the AWS Load Balancer Controller, deploys the 2048 sample app on Fargate, exposes it via Ingress backed by an ALB, verifies the deployment, and cleans up resources. 


## Prerequisites (what you need before starting)

- **AWS account** with permissions to create EKS, IAM, CloudFormation, EC2, VPC, and related resources.  
- **AWS CLI v2** installed and configured (`aws configure`).  
- **eksctl** installed.  
- **kubectl** installed.  
- **Helm 3** installed.  
- **WSL2 / Ubuntu** recommended for running commands (or any Linux/macOS shell).  
- **Local git** and a GitHub repo to push documentation and screenshots.

---

## Why these steps (short rationale)

- **Fargate profile**: lets pods run on AWS Fargate (serverless compute) so you can run pods without managing EC2 nodes. Useful for demos and cost control.  
- **OIDC provider**: allows Kubernetes service accounts to assume IAM roles securely .`eksctl utils associate-iam-oidc-provider` creates the OIDC provider for the cluster.  
- **IAM policy + IAM service account**: the AWS Load Balancer Controller needs specific AWS permissions to create and manage ALBs. We attach a least‑privilege policy to a Kubernetes service account via IRSA so the controller can act with those permissions.  
- **Helm**: Helm simplifies installing and managing the AWS Load Balancer Controller (templating, upgrades, values). It’s the recommended installation method for the controller.  
- **Ingress → ALB**: the controller watches Ingress resources and provisions an ALB with target groups to route external traffic to your pods.

---


## Step‑by‑step commands 

> Run from Ubuntu / WSL2** (recommended). Replace `<ACCOUNT_ID>`, `<REGION>`, `<VPC_ID>` as needed.



1.aws configure
> enter Access Key ID, Secret Access Key, default region (e.g., us-east-1), output json

2. Create EKS cluster

> eksctl create cluster --name demo-cluster --region us-east-1 --nodes 2 --node-type t3.micro
#create a Fargate-only cluster if you prefer serverless pods

3. Create a Fargate profile (optional — used in demo)

> eksctl create fargateprofile \
  --cluster demo-cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
This ensures pods in game-2048 run on Fargate.

4. Deploy sample app (Ingress + Service + Deployment)

> kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
#output: namespace/game-2048 created, deployment/service/ingress created
5. Check Ingress address (initially may be empty)

> kubectl get ingress -n game-2048
#If ADDRESS is empty, the ALB controller is not yet ready or lacks permissions
6. If no ADDRESS, ensure controller has permissions (OIDC + IAM policy + service account)
a) Associate OIDC provider (creates OIDC for IRSA)

> eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
Why: this creates the IAM OIDC identity provider for the cluster so Kubernetes service accounts can assume IAM roles.

b) Download the controller IAM policy

> curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
c) Create the IAM policy

> aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "PolicyId": "ANPAYHJAWRI7QPRK52KYZ",
        "Arn": "arn:aws:iam::565393656383:policy/AWSLoadBalancerControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2026-08-12T10:21:41+00:00",
        "UpdateDate": "2026-08-12T10:21:41+00:00"
    }
}

d) Create IAM service account (IRSA)

> eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
Why: this creates a Kubernetes service account bound to an IAM role with the controller policy. The controller pod uses this service account to call AWS APIs securely.

7. Install AWS Load Balancer Controller with Helm
bash
> helm repo add eks https://aws.github.io/eks-charts
> helm repo update

>helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<VPC_ID>
Why Helm: Helm handles templating and makes upgrades/rollbacks easy. We set serviceAccount.create=false because we already created the IRSA service account.

8. Verify controller deployment
bash
>kubectl get deployment -n kube-system aws-load-balancer-controller
>kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller -o wide
Wait until READY shows 2/2 and pods are Running.

9. Re-check Ingress and ALB address
bash
>kubectl get ingress -n game-2048
#ADDRESS should now show an ALB DNS like:
k8s-game2048-ingress2-xxxxxxxxxx.us-east-1.elb.amazonaws.com
10. Test the app
bash
open http://<ALB_DNS> in a browser to see the 2048 app UI



**Troubleshooting (common issues and fixes)**
Ingress ADDRESS remains empty

Ensure the AWS Load Balancer Controller pods are Running and Ready.

Confirm the IAM policy ARN is attached to the service account role and OIDC provider exists.

Confirm vpcId and region values passed to Helm are correct.



Cleanup **
**
> eksctl delete cluster --name demo-cluster --region us-east-1
#Also delete any IAM policies you created if not needed:
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy


Security notes & best practices
Never commit AWS credentials. Add .aws/ to .gitignore.

Use least‑privilege IAM policies; the iam_policy.json used here is the controller’s recommended policy.

Delete clusters and unused IAM resources after demos to avoid charges.
