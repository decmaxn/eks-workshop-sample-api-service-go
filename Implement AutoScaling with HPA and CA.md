In this Chapter, we will show patterns for scaling your worker nodes and applications deployments automatically. Automatic scaling in K8s comes in two forms:

1. Horizontal Pod Autoscaler (HPA) scales the pods in a deployment or replica set. It is implemented as a K8s API resource and a controller. The controller manager queries the resource utilization against the metrics specified in each HorizontalPodAutoscaler definition. It obtains the metrics from either the resource metrics API (for per-pod resource metrics), or the custom metrics API (for all other metrics).

1. Cluster Autoscaler (CA) is the default K8s component that can be used to perform pod scaling as well as scaling nodes in a cluster. It automatically increases the size of an Auto Scaling group so that pods have a place to run. And it attempts to remove idle nodes, that is, nodes with no running pods.

# Configure Horizontal Pod AutoScaler (HPA)
## Deploy the Metrics Server

Metrics Server is a cluster-wide aggregator of resource usage data. These metrics will drive the scaling behavior of the deployments. We will deploy the metrics server using Helm configured in a previous module
```
helm install stable/metrics-server \
    --name metrics-server \
    --version 2.0.4 \
    --namespace metrics
```
## Confirm the Metrics API is available.

Return to the terminal in the Cloud9 Environment
```
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```
If all is well, you should see a status message similar to the one below in the response
```
status:
  conditions:
  - lastTransitionTime: 2018-10-15T15:13:13Z
    message: all checks passed
    reason: Passed
    status: "True"
    type: Available
```
## We are now ready to scale a deployed application

# Scale an Application with HPA
## Deploy a Sample App

We will deploy an application and expose as a service on TCP port 80. The application is a custom-built image based on the php-apache image. The index.php page performs calculations to generate CPU load. More information can be found here
```
$ kubectl run php-apache --image=k8s.gcr.io/hpa-example --requests=cpu=200m --expose --port=80
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
service/php-apache created
deployment.apps/php-apache created
```
## Create an HPA resource

This HPA scales up when CPU exceeds 50% of the allocated container resource.
```
$ kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
horizontalpodautoscaler.autoscaling/php-apache autoscaled
```
View the HPA using kubectl. You probably will see <unknown>/50% for 1-2 minutes and then you should be able to see 0%/50%
```
kubectl get hpa
```
## Generate load to trigger scaling

Open a new terminal in the Cloud9 Environment and run the following command to drop into a shell on a new container
```
kubectl run -i --tty load-generator --image=busybox /bin/sh
```
Execute a while loop to continue getting http:///php-apache
```
$ kubectl run -i --tty load-generator --image=busybox /bin/sh
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you don't see a command prompt, try pressing enter.
/ #
```
In the previous tab, watch the HPA with the following command
```
kubectl get hpa -w
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK.....
```
You will see HPA scale the pods from 1 up to our configured maximum (10) until the CPU average is below our target (50%)
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ kubectl get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          2m32s
php-apache   Deployment/php-apache   252%/50%   1     10    1     2m45s
php-apache   Deployment/php-apache   252%/50%   1     10    4     3m
php-apache   Deployment/php-apache   252%/50%   1     10    6     3m15s
php-apache   Deployment/php-apache   74%/50%   1     10    6     3m45s
php-apache   Deployment/php-apache   79%/50%   1     10    6     4m45s
php-apache   Deployment/php-apache   79%/50%   1     10    10    5m
php-apache   Deployment/php-apache   44%/50%   1     10    10    5m45s
php-apache   Deployment/php-apache   47%/50%   1     10    10    6m45s
```
You can now stop (Ctrl + C) load test that was running in the other terminal. You will notice that HPA will slowly bring the replica count to min number based on its configuration. You should also get out of load testing application by pressing Ctrl + D


# Configure Cluster Autoscaler (CA)

Cluster Autoscaler for AWS provides integration with Auto Scaling groups. It enables users to choose from four different options of deployment:

    One Auto Scaling group - This is what we will use
    Multiple Auto Scaling groups
    Auto-Discovery
    Master Node setup

## Configure the Cluster Autoscaler (CA)

We have provided a manifest file to deploy the CA. Copy the commands below into your Cloud9 Terminal.
```
mkdir ~/environment/cluster-autoscaler
cd ~/environment/cluster-autoscaler
wget https://eksworkshop.com/scaling/deploy_ca.files/cluster_autoscaler.yml
```
## Configure the ASG

We will need to provide the name of the Autoscaling Group that we want CA to manipulate. Collect the name of the Auto Scaling Group (ASG) containing your worker nodes. Record the name somewhere. We will use this later in the manifest file.

You can find it in the console by following this link.

ASG

Check the box beside the ASG and click Actions and Edit

Change the following settings:

    Min: 2
    Max: 8

ASG Config

Click Save
## Configure the Cluster Autoscaler

Using the file browser on the left, open cluster_autoscaler.yml

Search for command: and within this block, replace the placeholder text <AUTOSCALING GROUP NAME> with the ASG name that you copied in the previous step. Also, update AWS_REGION value to reflect the region you are using and Save the file.
```
command:
  - ./cluster-autoscaler
  - --v=4
  - --stderrthreshold=info
  - --cloud-provider=aws
  - --skip-nodes-with-local-storage=false
  - --nodes=2:8:eksctl-eksworkshop-eksctl-nodegroup-ng-a35767d0-NodeGroup-S2Q0ATWB282P
env:
  - name: AWS_REGION
    value: us-east-2
```
This command contains all of the configuration for the Cluster Autoscaler. The primary config is the --nodes flag. This specifies the minimum nodes (2), max nodes (8) and ASG Name.

Although Cluster Autoscaler is the de facto standard for automatic scaling in K8s, it is not part of the main release. We deploy it like any other pod in the kube-system namespace, similar to other management pods.
## Create an IAM Policy

We need to configure an inline policy and add it to the EC2 instance profile of the worker nodes

Ensure ROLE_NAME is set in your environment: (this $ROLE_NAME is set in ~/.bash_profile earlier)
```
test -n "$ROLE_NAME" && echo ROLE_NAME is "$ROLE_NAME" || echo ROLE_NAME is not set
```
If ROLE_NAME is not set, please review: "Launch using eksctl.md" -"Test the cluster"
```
mkdir ~/environment/asg_policy
cat <<EoF > ~/environment/asg_policy/k8s-asg-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": "*"
    }
  ]
}
EoF
aws iam put-role-policy --role-name $ROLE_NAME --policy-name ASG-Policy-For-Worker --policy-document file://~/environment/asg_policy/k8s-asg-policy.json
```
This IAM policy has been created for you and has been attached to the correct role.
You can proceed with the next step.

Validate that the policy is attached to the role
```
$ aws iam get-role-policy --role-name $ROLE_NAME --policy-name ASG-Policy-For-Worker
{
    "RoleName": "eksctl-eksworkshop-eksctl-nodegro-NodeInstanceRole-DRUOI5R61EVT", 
    "PolicyDocument": {
        "Version": "2012-10-17", 
        "Statement": [
            {
                "Action": [
                    "autoscaling:DescribeAutoScalingGroups", 
                    "autoscaling:DescribeAutoScalingInstances", 
                    "autoscaling:SetDesiredCapacity", 
                    "autoscaling:TerminateInstanceInAutoScalingGroup"
                ], 
                "Resource": "*", 
                "Effect": "Allow"
            }
        ]
    }, 
    "PolicyName": "ASG-Policy-For-Worker"
}
```
## Deploy the Cluster Autoscaler
```
$ kubectl apply -f ~/environment/cluster-autoscaler/cluster_autoscaler.yml
serviceaccount/cluster-autoscaler created
clusterrole.rbac.authorization.k8s.io/cluster-autoscaler created
role.rbac.authorization.k8s.io/cluster-autoscaler created
clusterrolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
rolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
deployment.extensions/cluster-autoscaler created
```
Watch the logs
```
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```
## We are now ready to scale our cluster

# Scale a Cluster with CA
## Deploy a Sample App

We will deploy an sample nginx application as a ReplicaSet of 1 Pod
````
cat <<EoF> ~/environment/cluster-autoscaler/nginx.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-to-scaleout
spec:
  replicas: 1
  template:
    metadata:
      labels:
        service: nginx
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-to-scaleout
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi
EoF
kubectl apply -f ~/environment/cluster-autoscaler/nginx.yaml
kubectl get deployment/nginx-to-scaleout
```
## Scale our ReplicaSet

OK, let’s scale out the replicaset to 10
```
kubectl scale --replicas=10 deployment/nginx-to-scaleout
```
Some pods will be in the Pending state, which triggers the cluster-autoscaler to scale out the EC2 fleet.
```
kubectl get pods -o wide --watch
```
```
NAME                                 READY     STATUS    RESTARTS   AGE

nginx-to-scaleout-7cb554c7d5-2d4gp   0/1       Pending   0          11s
nginx-to-scaleout-7cb554c7d5-2nh69   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-45mqz   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-4qvzl   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-5jddd   1/1       Running   0          34s
nginx-to-scaleout-7cb554c7d5-5sx4h   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-5xbjp   0/1       Pending   0          11s
nginx-to-scaleout-7cb554c7d5-6l84p   0/1       Pending   0          11s
nginx-to-scaleout-7cb554c7d5-7vp7l   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-86pr6   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-88ttw   0/1       Pending   0          12s
```
View the cluster-autoscaler logs
```
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

You will notice Cluster Autoscaler events similar to below CA Scale Up events

Check the AWS Management Console to confirm that the Auto Scaling groups are scaling up to meet demand. This may take a few minutes. You can also follow along with the pod deployment from the command line. You should see the pods transition from pending to running as nodes are scaled up.

# Cleanup Scaling
```
kubectl delete -f ~/environment/cluster-autoscaler/cluster_autoscaler.yml
kubectl delete -f ~/environment/cluster-autoscaler/nginx.yaml
kubectl delete hpa,svc php-apache
kubectl delete deployment php-apache load-generator
rm -rf ~/environment/cluster-autoscaler
```