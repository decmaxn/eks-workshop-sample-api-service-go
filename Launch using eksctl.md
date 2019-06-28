# Prerequisites
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv -v /tmp/eksctl /usr/local/bin
eksctl version
```

# Create an EKS cluster 
Use `AWS_REGION=us-east-2` to create the cluster in us-east-2 instead of us-west-2.
```
eksctl create cluster --version=1.13 --name=eksworkshop-eksctl --nodes=3 --node-ami=auto --region=${AWS_REGION}
[ℹ]  using region us-west-2                                                                      
[ℹ]  setting availability zones to [us-west-2b us-west-2c us-west-2a]                            
[ℹ]  subnets for us-west-2b - public:192.168.0.0/19 private:192.168.96.0/19                      
[ℹ]  subnets for us-west-2c - public:192.168.32.0/19 private:192.168.128.0/19                    [ℹ]  subnets for us-west-2a - public:192.168.64.0/19 private:192.168.160.0/19                    
[ℹ]  nodegroup "ng-dbdbf1e4" will use "ami-089d3b6350c1769a6" [AmazonLinux2/1.13]                
[ℹ]  creating EKS cluster "eksworkshop-eksctl" in "us-west-2" region                             
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial nodegroup   
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stac
ks --region=us-west-2 --name=eksworkshop-eksctl'                                                 
[ℹ]  2 sequential tasks: { create cluster control plane "eksworkshop-eksctl", create nodegroup "n
g-dbdbf1e4" }                                                                                    
[ℹ]  building cluster stack "eksctl-eksworkshop-eksctl-cluster"                                  
[ℹ]  deploying stack "eksctl-eksworkshop-eksctl-cluster"                                         
[ℹ]  building nodegroup stack "eksctl-eksworkshop-eksctl-nodegroup-ng-dbdbf1e4"                  [ℹ]  --nodes-min=3 was set automatically for nodegroup ng-dbdbf1e4                               
[ℹ]  --nodes-max=3 was set automatically for nodegroup ng-dbdbf1e4                               
[ℹ]  deploying stack "eksctl-eksworkshop-eksctl-nodegroup-ng-dbdbf1e4"                           
[✔]  all EKS cluster resource for "eksworkshop-eksctl" had been created                          
[✔]  saved kubeconfig as "/home/ec2-user/.kube/config"                                           
[ℹ]  adding role "arn:aws:iam::902032610137:role/eksctl-eksworkshop-eksctl-nodegro-NodeInstanceRo
le-G6E8DKDM3LZ5" to auth ConfigMap                                                               
[ℹ]  nodegroup "ng-dbdbf1e4" has 0 node(s)                                                       
[ℹ]  waiting for at least 3 node(s) to become ready in "ng-dbdbf1e4"                             
[ℹ]  nodegroup "ng-dbdbf1e4" has 3 node(s)                                                       [ℹ]  node "ip-192-168-25-65.us-west-2.compute.internal" is ready                                 
[ℹ]  node "ip-192-168-52-104.us-west-2.compute.internal" is ready                                [ℹ]  node "ip-192-168-76-117.us-west-2.compute.internal" is ready                                
[ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'     
[✔]  EKS cluster "eksworkshop-eksctl" in "us-west-2" region is ready    
```

## Configure IAM authentication for the cluster:

I don't get it this time, but >You will likely see an error in the output similar to this:
````
[✖]  neither aws-iam-authenticator nor heptio-authenticator-aws are installed
[ℹ]  cluster should be functional despite missing (or misconfigured) client binaries
[✔]  EKS cluster "eksworkshop-eksctl" in "us-east-2" region is ready
````
This is caused by eksctl wanting to use aws-iam-authencator to pull IAM tokens. As of aws-cli version 1.16.156, this token functionality is built in. Let’s check our aws-cli version:
````
aws --version # in Cloud9, the version output should already be higher than 1.16.156
aws-cli/1.16.102 Python/2.7.14 Linux/4.14.97-90.72.amzn2.x86_64 botocore/1.12.92
````
Your Cloud9 workspace should already have a version late enough to pull IAM tokens.

If you need to update the aws-cli, use this command:
````
pip install awscli --upgrade --user # this updates aws-cli to the latest available version for the user
aws --version
````
We can update our kubectl config file to use the aws-cli for pulling IAM tokens with this command:
````
aws eks update-kubeconfig --name eksworkshop-eksctl # this updates kubeconfig to pull iam tokens using aws-cli
````
## Test the cluster:

Confirm your Nodes:
```
kubectl get nodes # if we see our 3 nodes, we know we have authenticated correctly
NAME                                           STATUS   ROLES    AGE   VERSION                   
ip-192-168-25-65.us-west-2.compute.internal    Ready    <none>   16m   v1.13.7-eks-c57ff8        
ip-192-168-52-104.us-west-2.compute.internal   Ready    <none>   16m   v1.13.7-eks-c57ff8        
ip-192-168-76-117.us-west-2.compute.internal   Ready    <none>   16m   v1.13.7-eks-c57ff8  
````
Export the Worker Role Name for use throughout the workshop
````
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
INSTANCE_PROFILE_ARN=$(aws cloudformation describe-stacks --stack-name $STACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InstanceProfileARN") | .OutputValue')
ROLE_NAME=$(aws cloudformation describe-stacks --stack-name $STACK_NAME | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InstanceRoleARN") | .OutputValue' | cut -f2 -d/)
echo "export ROLE_NAME=${ROLE_NAME}" >> ~/.bash_profile
echo "export INSTANCE_PROFILE_ARN=${INSTANCE_PROFILE_ARN}" >> ~/.bash_profile
````
Congratulations!

You now have a fully working Amazon EKS Cluster that is ready to use!
```