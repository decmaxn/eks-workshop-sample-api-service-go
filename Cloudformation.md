Refer to howto-launch-eks-workshop\modules\ for more documents. 

# Launch Using CloudFormation

AWS Officially provides several resources for building EKS and supporting infrastructure.

In this Chapter, we will take a look at the official CloudFormation method of Launching EKS.

## Create the EKS Service Role

A service-linked role is a unique type of IAM role that is linked directly to an AWS service. Service-linked roles are predefined by the service and include all the permissions that the service requires to call other AWS services on your behalf.

First we need to create the EKS Service Role, an IAM Role that EKS will use to manage your EKS Cluster:
```
cd howto-launch-eks-workshop/
aws iam create-role --role-name "eks-service-role-workshop" \
--assume-role-policy-document file://assets/eks-iam-trust-policy.json \
--description "EKS Service role created through the HOWTO workshop"
```

## Attach IAM Policy to the EKS Service Role

We have now created the IAM Role for the EKS Service. At the moment, it still has no permissions and can’t do anything. We can confirm that with this command:

aws iam list-attached-role-policies --role-name "eks-service-role-workshop"

We need to attach IAM Policies to the role to grant it permissions.

EKS has pre-created two policies that we can use:
```
aws iam attach-role-policy --role-name "eks-service-role-workshop" --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

aws iam attach-role-policy --role-name "eks-service-role-workshop" --policy-arn arn:aws:iam::aws:policy/AmazonEKSServicePolicy
```
Expect to see no output from these two commands. Only failure of the commands will produce an output.

You now have the IAM Role and Permissions required to Launch an EKS cluster.
## Create the VPC

EKS suggests launching a single EKS Cluster into a VPC. To make that easy, EKS provides a CloudFormation template that we can use:
```
aws cloudformation create-stack --stack-name "eksworkshop-cf" --template-url "https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/amazon-eks-vpc-sample.yaml"
```
The creation of the EKS VPC Stack will take about 2 minutes.

This is a script that will let you know when the CloudFormation stack is complete:
```
until [[ `aws cloudformation describe-stacks --stack-name "eksworkshop-cf" --query "Stacks[0].[StackStatus]" --output text` == "CREATE_COMPLETE" ]]; do  echo "The stack is NOT in a state of CREATE_COMPLETE at `date`";   sleep 30; done && echo "The Stack is built at `date` - Please proceed"
```
Now that you have your VPC ready, you can Launch your EKS cluster.

## Create the EKS Cluster

To build the EKS cluster, we need to tell the EKS service which IAM Service role to use, and which Subnets and Security Group to use. We can gather this information from our previous labs where we built the IAM role and VPC:
```
export SERVICE_ROLE=$(aws iam get-role --role-name "eks-service-role-workshop" --query Role.Arn --output text)

export SECURITY_GROUP=$(aws cloudformation describe-stacks --stack-name "eksworkshop-cf" --query "Stacks[0].Outputs[?OutputKey=='SecurityGroups'].OutputValue" --output text)

export SUBNET_IDS=$( aws cloudformation describe-stacks --stack-name "eksworkshop-cf" --query "Stacks[0].Outputs[?OutputKey=='SubnetIds'].OutputValue" --output text)
```
Let’s confirm the variables are now set in our environment:
```
echo SERVICE_ROLE=${SERVICE_ROLE}
echo SECURITY_GROUP=${SECURITY_GROUP}
echo SUBNET_IDS=${SUBNET_IDS}
```
Now we can create the EKS cluster:
```
aws eks create-cluster --name eksworkshop-cf --role-arn "${SERVICE_ROLE}" --resources-vpc-config subnetIds="${SUBNET_IDS}",securityGroupIds="${SECURITY_GROUP}"
```
Cluster provisioning usually takes less than 10 minutes.

You can query the status of your cluster with the following command:

aws eks describe-cluster --name "eksworkshop-cf" --query cluster.status --output text

When your cluster status is ACTIVE you can proceed.

## Create KubeConfig File

Our KubeConfig file will need to know specific info about our EKS cluster. We can gather that info and add it to our environment before building the file:
```
export EKS_ENDPOINT=$(aws eks describe-cluster --name eksworkshop-cf  --query cluster.[endpoint] --output=text)
export EKS_CA_DATA=$(aws eks describe-cluster --name eksworkshop-cf  --query cluster.[certificateAuthority.data] --output text)
```
Let’s confirm the variables are now set in our environment:
```
echo EKS_ENDPOINT=${EKS_ENDPOINT}
echo EKS_CA_DATA=${EKS_CA_DATA}
```
Now we can create the KubeConfig file:
```
mkdir ${HOME}/.kube

cat <<EoF > ${HOME}/.kube/config-eksworkshop-cf
  apiVersion: v1
  clusters:
  - cluster:
      server: ${EKS_ENDPOINT}
      certificate-authority-data: ${EKS_CA_DATA}
    name: kubernetes
  contexts:
  - context:
      cluster: kubernetes
      user: aws
    name: aws
  current-context: aws
  kind: Config
  preferences: {}
  users:
  - name: aws
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1alpha1
        command: heptio-authenticator-aws
        args:
          - "token"
          - "-i"
          - "eksworkshop-cf"
EoF
```
We now need to add this new config to the KubeCtl Config list:
```
export KUBECONFIG=${HOME}/.kube/config-eksworkshop-cf
echo "export KUBECONFIG=${KUBECONFIG}" >> ${HOME}/.bashrc
```
Let’s confirm that your KubeConfig is available:

kubectl config view

Let’s confirm that you can communicate with the Kubernetes API for your EKS cluster

kubectl get svc

## Add Workers

Now that your VPC and Kubernetes control plane are created, you can launch and configure your worker nodes. We will now use CloudFormation to launch worker nodes that will connect to the EKS cluster:
```
export SUBNET_IDS=$(aws cloudformation describe-stacks --stack-name "eksworkshop-cf" --query "Stacks[0].Outputs[?OutputKey=='SubnetIds'].OutputValue" --output text)
export SECURITY_GROUP=$(aws cloudformation describe-stacks --stack-name "eksworkshop-cf" --query "Stacks[0].Outputs[?OutputKey=='SecurityGroups'].OutputValue" --output text)
export VPC_ID=$(aws cloudformation describe-stacks --stack-name "eksworkshop-cf" --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text)

aws cloudformation create-stack --stack-name "eksworkshop-cf-worker-nodes" \
--template-url "https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/amazon-eks-nodegroup.yaml" \
--capabilities CAPABILITY_IAM \
--parameters \
ParameterKey=KeyName,ParameterValue=eksworkshop \
ParameterKey=NodeImageId,ParameterValue=ami-73a6e20b \
ParameterKey=NodeGroupName,ParameterValue=eks-howto-workers \
ParameterKey=ClusterControlPlaneSecurityGroup,ParameterValue=$SECURITY_GROUP \
ParameterKey=VpcId,ParameterValue=$VPC_ID \
ParameterKey=Subnets,ParameterValue=\"$SUBNET_IDS\" \
ParameterKey=ClusterName,ParameterValue=eksworkshop-cf
```
The creation of the workers will take about 3 minutes.
This is a script that will let you know when the CloudFormation stack is complete:
```
until [[ `aws cloudformation describe-stacks --stack-name "eksworkshop-cf-worker-nodes" --query "Stacks[0].[StackStatus]" --output text` == "CREATE_COMPLETE" ]]; do  echo "The stack is NOT in a state of CREATE_COMPLETE at `date`";   sleep 30; done && echo "The Stack is built at `date` - Please proceed"
```
## Create the Worker ConfigMap

The file found in the repository at assets/worker-configmap.yml contains a configmap we can use for our EKS workers. We need to substitute our instance role ARN into the template:

View the template:
``
cd ${HOME}/environment/howto-launch-eks-workshop/

cat assets/worker-configmap.yml
```
Lookup and store the Instance ARN:
```
export INSTANCE_ARN=$(aws cloudformation describe-stacks --stack-name "eksworkshop-cf-worker-nodes" --query "Stacks[0].Outputs[?OutputKey=='NodeInstanceRole'].OutputValue" --output text)

echo INSTANCE_ARN=$INSTANCE_ARN
```
Test modify the template to see what changes:
```
sed "s@.*rolearn.*@    - rolearn: $INSTANCE_ARN@" assets/worker-configmap.yml
```
Actually apply the configmap:
```
sed "s@.*rolearn.*@    - rolearn: $INSTANCE_ARN@" assets/worker-configmap.yml | kubectl apply -f /dev/stdin
```

## View the Node Status

You should now see the status of your nodes as they become ready.

kubectl get nodes --watch

Congratulations!

You now have a fully working Amazon EKS Cluster that is ready to use!