Continuous integration (CI) and continuous delivery (CD) are essential for deft organizations. Teams are more productive when they can make discrete changes frequently, release those changes programmatically and deliver updates without disruption.

In this module, we will build a CI/CD pipeline using AWS CodePipeline. The CI/CD pipeline will deploy a sample Kubernetes service, we will make a change to the GitHub repository and observe the automated delivery of this change to the cluster.

# Create IAM Role

In an AWS CodePipeline, we are going to use AWS CodeBuild to deploy a sample Kubernetes service. This requires an AWS Identity and Access Management (IAM) role capable of interacting with the EKS cluster.

In this step, we are going to create an IAM role and add an inline policy that we will use in the CodeBuild stage to interact with the EKS cluster via kubectl.

Create the role:

    Workshop at AWS event
    Workshop in your own account

```
cd ~/environment

TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-role-policy

aws iam create-role --role-name EksWorkshopCodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

aws iam put-role-policy --role-name EksWorkshopCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy
```

# Modify aws-auth ConfigMap

Now that we have the IAM role created, we are going to add the role to the aws-auth ConfigMap for the EKS cluster.

Once the ConfigMap includes this new role, kubectl in the CodeBuild stage of the pipeline will be able to interact with the EKS cluster via the IAM role.
```
ROLE="    - rolearn: arn:aws:iam::$ACCOUNT_ID:role/EksWorkshopCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"

kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml

kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
```
If you would like to edit the aws-auth ConfigMap manually, you can run: $ kubectl edit -n kube-system configmap/aws-auth

Me: It added the following mapRoles on top of existing groups
```
    - rolearn: arn:aws:iam:::role/EksWorkshopCodeBuildKubectlRole
      username: build
      groups:
        - system:masters
```
# Fork Sample Repository

We are now going to fork the sample Kubernetes service so that we will be able modify the repository and trigger builds.

Login to GitHub and fork the sample service to your own account:

https://github.com/rnzsgh/eks-workshop-sample-api-service-go

Me: This repo is the same as this repo where this file is in. 

# GitHub Access Token

In order for CodePipeline to receive callbacks from GitHub, we need to generate a personal access token.

Once created, an access token can be stored in a secure enclave and reused, so this step is only required during the first run or when you need to generate new keys.

Open up the New personal access page in GitHub.

You may be prompted to enter your GitHub password

Enter a value for Token description, check the repo permission scope and scroll down and click the Generate token button

Refer to https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line

Copy the personal access token and save it in a secure place for the next step
6583651a2d4f4a3931befa815760f01e53f39348

# CodePipeline Setup

Now we are going to create the AWS CodePipeline using AWS CloudFormation.

Each EKS deployment/service should have its own CodePipeline and be located in an isolated source repository.

You can modify the CloudFormation templates provided with this workshop to meet your system requirements to easily onboard new services to your EKS cluster. 

For each new service the following steps can be repeated.

Click the Launch button to create the CloudFormation stack in the AWS Management Console. [ci-cd-codepipeline.cfn.yml](https://s3.amazonaws.com/eksworkshop.com/templates/master/ci-cd-codepipeline.cfn.yml)	
CodePipeline & EKS 	

After the console is open, enter your GitHub username, personal access token (created in previous step), check the acknowledge box and then click the “Create stack” button located at the bottom of the page.
```
aws cloudformation create-stack  --stack-name eksws-codepipeline  \
    --template-body file://ci-cd-codepipeline.cfn.yml \
    --parameters  ParameterKey=GitHubUser,ParameterValue=decmaxn ParameterKey=GitHubToken,ParameterValue=6583651a2d4f4a3931befa815760f01e53f39348 \
    --capabilities CAPABILITY_IAM
```

Wait for the status to change from “CREATE_IN_PROGRESS” to CREATE_COMPLETE before moving on to the next step.

CloudFormation Stack

Open CodePipeline in the Management Console. You will see a CodePipeline that starts with eks-workshop-codepipeline. Click this link to view the details.

If you receive a permissions error similar to User x is not authorized to perform: codepipeline:ListPipelines… upon clicking the above link, the CodePipeline console may have opened up in the wrong region. To correct this, from the Region dropdown in the console, choose the region you provisioned the workshop in. Select Oregon (us-west-2) if you provisioned the workshow per the “Start the workshop at an AWS event” instructions.

[CodePipeline Landing picture]

Once you are on the detail page for the specific CodePipeline, you can see the status along with links to the change and build details.

[CodePipeline Details picture]

If you click on the “details” link in the build/deploy stage, you can see the output from the CodeBuild process.

To figure out why CodeBuild get "error: You must be logged in to the server (the server has asked for the client to provide credentials) " when run kubectl apply -f hello-k8s.yml, My added cat command shows:
```
[Container] 2019/07/01 18:28:09 Running command cat $HOME/.kube/config 
apiVersion: v1 
clusters: 
- cluster: 
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRFNU1EY3dNVEUwTWpneE5Wb1hEVEk1TURZeU9ERTBNamd4TlZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBT0JyCnIvR0JGRDJ0SDNONHZVMG9neXp3YW56WjZ5Y3RlNEJpaDFud0JGbWhKbThkVzFuckN3T25jNU1zOW5CSkRmaEoKNmFSTlJWdS9INWdYelRmbXRMQWwreUNlOTR1QkF5UmlCOUxEUStCcUYveDBZZVJJSllsRG1NM1cvc2pvMm5KSwpLUmtSQ2czd2QyRm95b2JhcENhdnlsVlhLV1IxWHJ2RnFTbXZYd2FTbWpPQkpzWE1TVEFDaGgvN1l4eTFJdjR1CjcwMFEyYUdrRUlaMnNsdytKSjdtZ01scTY0M1ZjMEZOa05LclVvQi9pOE1yQlhueXI5K1ZWMFVzck83Q3NVY2cKWkFLNVQydjE0K3hvK3ViRDBvL3d5SElSRTN6cmpEMjBLcTB5Q0xYU2t5Nm9iWkdjZnRUTG9ZaUw4bGpyMTNTSApvUzJpanF5V1V5RmQ5cmx6Q3djQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFLdW1WaFJKS2tHeVVENlM1bkhGcno4bXlWd04KY2lHOENUYmFYbStmeEQrSmZsbStwWlUrdU0va3REbHhROG0zTmFqdE5wTGlSNVpFOVF2RU1oSGY5RUp3cFR6VApOekF6VkJMa0s4QlQ0TVNKVmQ1ZldqY2hYQVNsbWxjRklMTSs1Q2ZDN1kvT2pxWDlBRDZUMlYxc3FlWmdaVU1ZCk1XSytwSVZQRTd6UWF1OG1nK2U5c0Z5QlhXK0ZVVWtsQ2tOTWJhN2FOUHlxYndwSTZRYzM3Y25vMzhjMElVcm8KVG5XaGFDa1YwRlNSeHBPc2VlRFZWOENLS0hMcnI4VFBRc2hrSytOc2JPWUNtakZDdDNlK1Bpc2NFMVRoeUZwNgoyY3RlZUJMSnpDWkxIN2REYzdOUmlncWdwT3dLMWMxMVhkelRxL2szcWkyemNwR21UTnFlUVFZK0lMcz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo= 
    server: https://4A42A72559C1D3ABFED5A6ADEA137F46.yl4.us-east-2.eks.amazonaws.com 
  name: arn:aws:eks:us-east-2:902032610137:cluster/eksworkshop-eksctl 
contexts: 
- context: 
    cluster: arn:aws:eks:us-east-2:902032610137:cluster/eksworkshop-eksctl 
    user: arn:aws:eks:us-east-2:902032610137:cluster/eksworkshop-eksctl 
  name: arn:aws:eks:us-east-2:902032610137:cluster/eksworkshop-eksctl 
current-context: arn:aws:eks:us-east-2:902032610137:cluster/eksworkshop-eksctl 
kind: Config 
preferences: {} 
users: 
- name: arn:aws:eks:us-east-2:902032610137:cluster/eksworkshop-eksctl 
  user: 
    exec: 
      apiVersion: client.authentication.k8s.io/v1alpha1 
      args: 
      - --region 
      - us-east-2 
      - eks 
      - get-token 
      - --cluster-name 
      - eksworkshop-eksctl 
      command: aws 
```
And previous "Modify aws-auth ConfigMap" step has made this change for EksWorkshopCodeBuildKubectlRole to configmap. 
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ kubectl describe -n kube-system configmap/aws-auth                                                                                                             
Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
mapRoles:
----
- rolearn: arn:aws:iam:::role/EksWorkshopCodeBuildKubectlRole
  username: build
  groups:
    - system:masters
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws:iam::902032610137:role/eksctl-eksworkshop-eksctl-nodegro-NodeInstanceRole-1LBX25G43BGPV
  username: system:node:{{EC2PrivateDNSName}}

mapUsers:
----
```
Also I have noticed the EksWorkshopCodeBuildKubectlRole is not directly assumed by codebuild service, but indirectly by eksws-codepipeline-CodeBuildServiceRole-9ASHWYIPVHJR. 

Should this be the problem? I didn't find answer, and decide to continue. 
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ aws iam get-role --role-name EksWorkshopCodeBuildKubectlRole
{
    "Role": {
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17", 
            "Statement": [
                {
                    "Action": "sts:AssumeRole", 
                    "Effect": "Allow", 
                    "Principal": {
                        "AWS": "arn:aws:iam::902032610137:role/eksws-codepipeline-CodeBuildServiceRole-9ASHWYIPVHJR"
                    }
...
[ec2-user@bastion eks-workshop-sample-api-service-go]$ aws iam get-role --role-name eksws-codepipeline-CodeBuildServiceRole-9ASHWYIPVHJR
{
    "Role": {
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17", 
            "Statement": [
                {
                    "Action": "sts:AssumeRole", 
                    "Effect": "Allow", 
                    "Principal": {
                        "Service": "codebuild.amazonaws.com"
                    }
                }
```

To review the status of the deployment, you can run:
```
kubectl describe deployment hello-k8s
```
For the status of the service, run the following command:
```
kubectl describe service hello-k8s
```
Challenge:

How can we view our exposed service?

HINT: Which kubectl command will get you the Elastic Load Balancer (ELB) endpoint for this app?
Expand here to see the solution

Once the service is built and delivered, we can run the following command to get the Elastic Load Balancer (ELB) endpoint and open it in a browser. If the message is not updated immediately, give Kubernetes some time to deploy the change.
```
 kubectl get services hello-k8s -o wide
```
This service was configured with a LoadBalancer so, an AWS Elastic Load Balancer is launched by Kubernetes for the service. The EXTERNAL-IP column contains a value that ends with “elb.amazonaws.com” - the full value is the DNS address.
