## NodeJS Deployment to AWS EKS

### Prereq: CLI Tools
<details>
<summary>AWS CLI</summary>
<br>

```
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
```

```
sudo installer -pkg AWSCLIV2.pkg -target /
```

```
aws --version
```
</details>

<details>
<summary>kubectl</summary>
<br>

```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.20.4/2021-04-12/bin/darwin/amd64/kubectl
```
```
chmod +x ./kubectl 
```
```
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
```
```
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bash_profile
```
```
kubectl version --short --client
```
</details>

<details>
<summary>eksctl</summary>
<br>

```
brew install weaveworks/tap/eksctl
```
or instead update:
```
brew upgrade eksctl && brew link --overwrite eksctl
```
```
eksctl version 
```
</details>
<br>

### Setup
[IAM requirements docs](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role) for worker nodes
**IAM > Create Role > EC2 > Next: Permissions >** check the boxes for these policies: 
- `AmazonEC2ContainerRegistryReadOnly`
- `AmazonEKSWorkerNodePolicy`
- `AmazonEKS_CNI_Policy` 

Create EKS Cluster, then create node group only once cluster is `ACTIVE`, then verify worker nodes with:
```
kubectl get nodes
```

Create AWS ECR repo: `ex-app`

Replace `<aws_account_id>` and `<region>` below:
```
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
```
Expected Output: `Login Succeeded`

```
docker build -t ex-app .
```

```
docker tag eks-app:latest <aws_account_id>.dkr.ecr.<region>.amazonaws.com/ex-app:latest
```
```
docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/ex-app:latest
```

```
aws eks --region <region> update-kubeconfig --name <ex-app>
```
```
kubectl get nodes
```
```
kubectl create -f ./release
```
To get the **EXTERNAL IP**:
```
kubectl get nodes  -o wide
```
- Modify Node security group's inbound rules by adding a custom TCP port `31479` (as defined in the `service-manifest.yaml`) with `0.0.0.0/0` as the source  
- Now visit the listed `<External-IP>:31479` to view the basic app
    - Valid NodePort range: 30000 - 32767

## Teardown
```
kubectl delete deployments,svc --all 
```

```
aws eks list-nodegroups --cluster-name <ex-app>
```

```
aws eks delete-nodegroup --nodegroup-name <node-group-workers> --cluster-name <ex-app>
```

```
eksctl delete cluster --region=<region> --name=<ex-app>
```
