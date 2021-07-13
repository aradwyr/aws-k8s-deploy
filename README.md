## NodeJS in AWS EKS Deployment

Command Line Tools: 
```
$ curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
$ sudo installer -pkg AWSCLIV2.pkg -target /
$ aws --version
aws-cli/2.2.18 Python/3.8.8 Darwin/20.4.0 exe/x86_64 prompt/off

# install kubectl 
$ curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.20.4/2021-04-12/bin/darwin/amd64/kubectl
$ chmod +x ./kubectl 
$ mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
$ echo 'export PATH=$PATH:$HOME/bin' >> ~/.bash_profile
$ kubectl version --short --client
# Client Version: v1.20.4-eks-6b7464

# install eksctl
$ brew install weaveworks/tap/eksctl

# or instead update eksctl 
$ brew upgrade eksctl && brew link --overwrite eksctl
$ eksctl version 
# 0.56.0
```
- [IAM requirements docs](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role) for worker nodes
**IAM > Create Role > EC2 > Next: Permissions >** check the boxes for both of these policies: 
    - `AmazonEC2ContainerRegistryReadOnly`
    - `AmazonEKSWorkerNodePolicy`
    - `AmazonEKS_CNI_Policy` 
- Create EKS Cluster, then verify worker nodes with:
```
$ kubectl get nodes
```
- Create AWS ECR repo: eks-app 
- Replace <aws_account_id> and <region>
```
$ aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
# Login Succeeded

$ docker build -t eks-app .

$ docker tag eks-app:latest <aws_account_id>.dkr.ecr.<region>.amazonaws.com/eks-app:latest

$ docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/eks-app:latest

$ aws eks --region <region> update-kubeconfig --name <ex-app>
$ kubectl get nodes
$ kubectl create -f deploy-manifest.yaml
$ kubectl create -f service-manifest.yaml 
$ kubectl get nodes  -o wide
# Take note of the external IPs
```
- Modify Node security group's inbound rules by adding a custom TCP port `31479` (as defined in the `service-manifest.yaml`) with `0.0.0.0/0` as the source  
- Now visit the listed `<External-IP>:31479` to view the basic app
    - Valid NodePort range: 30000 - 32767

## Teardown
```
kubectl delete deployments,svc --all 
eksctl delete cluster --region=<region> --name=<ex-app>
```