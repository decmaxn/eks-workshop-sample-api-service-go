In this module, you will learn how to provision, manage, and maintain your Kubernetes clusters with Amazon EKS at any scale on Spot Instances to optimize cost and scale.

![combines purchase options, instance types and AZs in a single ASG](file://spot-diagram.png)

# Add EC2 Workers - On-Demand and Spot

We have our EKS Cluster and worker nodes already, but we need some Spot Instances configured as workers. We also need a Node Labeling strategy to identify which instances are Spot and which are on-demand so that we can make more intelligent scheduling decisions. We will use AWS CloudFormation to launch new worker nodes that will connect to the EKS cluster.

This template will create a single ASG that leverages the latest feature to mix multiple instance types and purchase as a single K8s nodegroup. Check out this blog: New – EC2 Auto Scaling Groups With Multiple Instance Types & Purchase Options for details.
## Retrieve the Worker Node Instance Profile ARN

First, we will need to ensure the ARN Name our workers use is set in our environment:
```
test -n "$INSTANCE_PROFILE_ARN" && echo INSTANCE_PROFILE_ARN is "$INSTANCE_PROFILE_ARN" || echo INSTANCE_PROFILE_ARN is not set
```
Copy the Profile ARN for use as a Parameter in the next step. If you receive an error or empty response, expand the steps below to export.
Expand here if you need to export the Instance Profile ARN
If INSTANCE_PROFILE_ARN is not set, please review: "Launch using eksctl.md" -"Test the cluster"
```
# Example Output
INSTANCE_PROFILE_ARN is arn:aws:iam::902032610137:instance-profile/eksctl-eksworkshop-eksctl-nodegroup-ng-a35767d0-NodeInstanceProfile-LZ4R7OKS8NNI
```
## Retrieve the Security Group Name

We also need to collect the ID of the security group used with the existing worker nodes.
```
STACK_NAME=$(aws cloudformation describe-stacks | jq -r '.Stacks[].StackName' | grep eksctl-eksworkshop-eksctl-nodegroup)
SG_ID=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME --logical-resource-id SG | jq -r '.StackResources[].PhysicalResourceId')
echo $SG_ID
```
```
# Example Output
$ echo $SG_ID
sg-0a197f7c3bba4a9b7
```
## Launch the CloudFormation Stack

We will launch the CloudFormation template as a new set of worker nodes, but it’s also possible to update the nodegroup CloudFormation stack created by the eksctl tool.

Click the Launch button to create the CloudFormation stack in the AWS Management Console.
[EKS Workers - Spot and On Demand](https://s3.amazonaws.com/eksworkshop.com/templates/master/amazon-eks-nodegroup-with-mixed-instances.yml) 	

Confirm the region is correct based on where you’ve deployed your cluster.
Once the console is open you will need to configure the missing parameters. Use the table below for guidance.
```
Parameter 	Value
Stack Name: 	eksworkshop-spot-workers
Cluster Name: 	eksworkshop-eksctl (or whatever you named your cluster)
ClusterControlPlaneSecurityGroup: 	Select from the dropdown. It will contain your cluster name and the words ‘ControlPlaneSecurityGroup’
NodeInstanceProfile: 	Use the Instance Profile ARN that copied in the step above. (e.g.eks-workshop-nodegroup)
UseExistingNodeSecurityGroups: 	Leave as ‘Yes’
ExistingNodeSecurityGroups: 	Use the SG name that copied in the step above. (e.g. sg-0123456789abcdef)
```
NodeImageId: 	Visit this [link](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html) and select the non-GPU image for your region - Check for empty spaces in copy/paste
```
KeyName: 	SSH Key Pair created earlier or any valid key will work
NodeGroupName: 	Leave as spotworkers
VpcId: 	Select your workshop VPC from the dropdown
Subnets: 	Select the 3 private subnets for your workshop VPC from the dropdown
BootstrapArgumentsForOnDemand: 	--kubelet-extra-args --node-labels=lifecycle=OnDemand
BootstrapArgumentsForSpotFleet: 	--kubelet-extra-args '--node-labels=lifecycle=Ec2Spot --register-with-taints=spotInstance=true:PreferNoSchedule'
```
## What’s going on with Bootstrap Arguments?

The EKS Bootstrap.sh script is packaged into the EKS Optimized AMI that we are using, and only requires a single input, the EKS Cluster name. The bootstrap script supports setting any kubelet-extra-args at runtime. We have configured node-labels so that kubernetes knows what type of nodes we have provisioned. We set the lifecycle for the nodes as OnDemand or Ec2Spot. We are also tainting with PreferNoSchedule to prefer pods not be scheduled on Spot Instances. This is a “preference” or “soft” version of NoSchedule – the system will try to avoid placing a pod that does not tolerate the taint on the node, but it is not required.
```
            ilc=`aws ec2 describe-instances --instance-ids  $iid  --query 'Reservations[0].Instances[0].InstanceLifecycle' --output text`
            if [ "$ilc" == "spot" ]; then
              /etc/eks/bootstrap.sh ${ClusterName} ${BootstrapArgumentsForSpotFleet}
            else
              /etc/eks/bootstrap.sh ${ClusterName} ${BootstrapArgumentsForOnDemand}
            fi
            # /etc/eks/bootstrap.sh ${ClusterName} $BootstrapArgumentsForOnDemand
```
You can leave the rest of the default parameters as is and continue through the remaining CloudFormation screens. Check the box next to I acknowledge that AWS CloudFormation might create IAM resources and click Create

The creation of the workers will take about 3 minutes.
## Confirm the Nodes

Confirm that the new nodes joined the cluster correctly. You should see 2-3 more nodes added to the cluster.
```
kubectl get nodes
```
All Nodes You can use the node-labels to identify the lifecycle of the nodes
```
kubectl get nodes --show-labels --selector=lifecycle=Ec2Spot
```
The output of this command should return 2 nodes. At the end of the node output, you should see the node label lifecycle=Ec2Spot
```
$ kubectl get nodes --show-labels --selector=lifecycle=Ec2Spot
NAME                                            STATUS   ROLES    AGE     VERSION              LABELS
ip-192-168-125-122.us-east-2.compute.internal   Ready    <none>   3m26s   v1.13.7-eks-c57ff8   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=c4.large,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=us-east-2,failure-domain.beta.kubernetes.io/zone=us-east-2c,kubernetes.io/hostname=ip-192-168-125-122.us-east-2.compute.internal,lifecycle=Ec2Spot
ip-192-168-184-9.us-east-2.compute.internal     Ready    <none>   3m20s   v1.13.7-eks-c57ff8   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=c4.large,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=us-east-2,failure-domain.beta.kubernetes.io/zone=us-east-2a,kubernetes.io/hostname=ip-192-168-184-9.us-east-2.compute.internal,lifecycle=Ec2Spot
```
Now we will show all nodes with the lifecycle=OnDemand. The output of this command should return 1 node as configured in our CloudFormation template.
```
$ kubectl get nodes --show-labels --selector=lifecycle=OnDemand
NAME                                           STATUS   ROLES    AGE     VERSION              LABELS
ip-192-168-156-48.us-east-2.compute.internal   Ready    <none>   5m21s   v1.13.7-eks-c57ff8   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=m4.large,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=us-east-2,failure-domain.beta.kubernetes.io/zone=us-east-2b,kubernetes.io/hostname=ip-192-168-156-48.us-east-2.compute.internal,lifecycle=OnDemand
```
You can use the kubectl describe nodes with one of the spot nodes to see the taints applied to the EC2 Spot Instances.
```
$ kubectl describe nodes ip-192-168-125-122.us-east-2.compute.interna
Name:               ip-192-168-125-122.us-east-2.compute.internal
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=c4.large
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=us-east-2
                    failure-domain.beta.kubernetes.io/zone=us-east-2c
                    kubernetes.io/hostname=ip-192-168-125-122.us-east-2.compute.internal
                    lifecycle=Ec2Spot
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sun, 30 Jun 2019 03:49:37 +0000
Taints:             spotInstance=true:PreferNoSchedule
Unschedulable:      false
```

# Deploy The Spot Interrupt Handler

In this section, we will prepare our cluster to handle Spot interruptions.

If the available On-Demand capacity of a particular instance type is depleted, the Spot Instance is sent an interruption notice two minutes ahead to gracefully wrap up things. We will deploy a pod on each spot instance to detect and redeploy applications elsewhere in the cluster

The first thing that we need to do is deploy the Spot Interrupt Handler on each Spot Instance. This will monitor the EC2 metadata service on the instance for a interruption notice.

The workflow can be summarized as:

    Identify that a Spot Instance is being reclaimed.
    Use the 2-minute notification window to gracefully prepare the node for termination.
    Taint the node and cordon it off to prevent new pods from being placed.
    Drain connections on the running pods.
    Replace the pods on remaining nodes to maintain the desired capacity.

We have provided an example K8s DaemonSet manifest. A DaemonSet runs one pod per node.
```
mkdir ~/environment/spot
cd ~/environment/spot
wget https://eksworkshop.com/spot/managespot/deployhandler.files/spot-interrupt-handler-example.yml
```
As written, the manifest will deploy pods to all nodes including On-Demand, which is a waste of resources. We want to edit our DaemonSet to only be deployed on Spot Instances. Let’s use the labels to identify the right nodes.

Use a nodeSelector to constrain our deployment to spot instances. View this link for more details.
Challenge

Configure our Spot Handler to use nodeSelector
Expand here to see the solution
Place this at the end of the DaemonSet manifest under Spec.Template.Spec.nodeSelector
```
      nodeSelector:
        lifecycle: Ec2Spot
```
Deploy the DaemonSet
```
$ kubectl apply -f ~/environment/spot/spot-interrupt-handler-example.yml
clusterrole.rbac.authorization.k8s.io/spot-interrupt-handler created
serviceaccount/spot-interrupt-handler created
clusterrolebinding.rbac.authorization.k8s.io/spot-interrupt-handler created
daemonset.apps/spot-interrupt-handler created
```
If you receive an error deploying the DaemonSet, there is likely a small error in the YAML file. We have provided a solution file at the bottom of this page that you can use to compare.

View the pods. There should be one for each spot node.
```
$ kubectl get daemonsets
NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR       AGE
spot-interrupt-handler   2         2         2       2            2           lifecycle=Ec2Spot   52s
```

Related files
spot-interrupt-handler-example.yml (1 ko)
spot-interrupt-handler-solution.yml (1 ko) 

# Deploy an Application on Spot

We are redesigning our Microservice example and want our frontend service to be deployed on Spot Instances when they are available. We will use Node Affinity in our manifest file to configure this.
## Configure Node Affinity and Tolerations

Open the deployment manifest in your Cloud9 editor - ~/environment/ecsdemo-frontend/kubernetes/deployment.yaml

Edit the spec to configure NodeAffinity to prefer Spot Instances, but not require them. This will allow the pods to be scheduled on On-Demand nodes if no spot instances were available or correctly labelled.

We also want to configure a toleration which will allow the pods to “tolerate” the taint that we configured on our EC2 Spot Instances.

For examples of Node Affinity, check this https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity

For examples of Taints and Tolerations, check this https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/
## Challenge

Configure Affinity and Toleration
Expand here to see the solution

Add this to your deployment file under spec.template.spec
```
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: lifecycle
                operator: In
                values:
                - Ec2Spot
      tolerations:
      - key: "spotInstance"
        operator: "Equal"
        value: "true"
        effect: "PreferNoSchedule"
```
We have provided a solution file below that you can use to compare.

Related files
deployment-solution.yml (1 ko)
## Redeploy the Frontend on Spot

First let’s take a look at all pods deployed on Spot instances
```
 for n in $(kubectl get nodes -l lifecycle=Ec2Spot --no-headers | cut -d " " -f1); do kubectl get pods --all-namespaces  --no-headers --field-selector spec.nodeName=${n} ; done
```
Now we will redeploy our microservices with our edited Frontend Manifest
```
cd ~/ecsdemo-frontend
kubectl apply -f kubernetes/service.yaml
kubectl apply -f kubernetes/deployment.yaml

cd ~/ecsdemo-crystal
kubectl apply -f kubernetes/service.yaml
kubectl apply -f kubernetes/deployment.yaml

cd ~/ecsdemo-nodejs
kubectl apply -f kubernetes/service.yaml
kubectl apply -f kubernetes/deployment.yaml
```
We can again check all pods deployed on Spot Instances and should now see the frontend pods running on Spot instances
```
for n in $(kubectl get nodes -l lifecycle=Ec2Spot --no-headers | cut -d " " -f1); do kubectl get pods --all-namespaces  --no-headers --field-selector spec.nodeName=${n} ; done
```

# Cleanup

Cleanup our Microservices deployment
```
cd ~/ecsdemo-frontend
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml

cd ~/ecsdemo-crystal
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml

cd ~/ecsdemo-nodejs
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
```
Cleanup the Spot Handler Daemonset
```
kubectl delete -f ~/environment/spot/spot-interrupt-handler-example.yml
```
To clean up the worker created by this module, run the following commands:

Remove the Worker nodes from EKS:
```
aws cloudformation delete-stack --stack-name "eksworkshop-spot-workers"
```