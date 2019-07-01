# Implement Logging with EFK

In this Chapter, we will deploy a common Kubernetes logging pattern which consists of the following:

    Fluentd is an open source data collector providing a unified logging layer, supported by 500+ plugins connecting to many types of systems.
    Elasticsearch is a distributed, RESTful search and analytics engine.
    Kibana lets you visualize your Elasticsearch data.

Together, Fluentd, Elasticsearch and Kibana is also known as “EFK stack”. Fluentd will forward logs from the individual instances in the cluster to a centralized logging backend (CloudWatch Logs) where they are combined for higher-level reporting using ElasticSearch and Kibana.

# Configure IAM Policy for Worker Nodes

We will be deploying Fluentd as a DaemonSet, or one pod per worker node. The fluentd log daemon will collect logs and forward to CloudWatch Logs. This will require the nodes to have permissions to send logs and create log groups and log streams. This can be accomplished with an IAM user, IAM role, or by using a tool like Kube2IAM.

In our example, we will create an IAM policy and attach it the the Worker node role.

First, we will need to ensure the Role Name our workers use is set in our environment:
```
test -n "$ROLE_NAME" && echo ROLE_NAME is "$ROLE_NAME" || echo ROLE_NAME is not set
```
If you receive an error or empty response, expand the steps below to export.
Expand here if you need to export the Role Name

    Workshop in your own account
````
mkdir ~/environment/iam_policy
cat <<EoF > ~/environment/iam_policy/k8s-logs-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "logs:DescribeLogGroups",
                "logs:DescribeLogStreams",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
EoF

aws iam put-role-policy --role-name $ROLE_NAME --policy-name Logs-Policy-For-Worker --policy-document file://~/environment/iam_policy/k8s-logs-policy.json
```
Validate that the policy is attached to the role
```
aws iam get-role-policy --role-name $ROLE_NAME --policy-name Logs-Policy-For-Worker
```

# Provision an Elasticsearch Cluster

This example creates a two instance Amazon Elasticsearch cluster named kubernetes-logs. This cluster is created in the same region as the Kubernetes cluster and CloudWatch log group.

Note that this cluster has an open access policy which will need to be locked down in production environments.
```
aws es create-elasticsearch-domain \
  --domain-name kubernetes-logs \
  --elasticsearch-version 6.3 \
  --elasticsearch-cluster-config \
  InstanceType=m4.large.elasticsearch,InstanceCount=2 \
  --ebs-options EBSEnabled=true,VolumeType=standard,VolumeSize=100 \
  --access-policies '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":["*"]},"Action":["es:*"],"Resource":"*"}]}'
```
It takes a little while for the cluster to be created and arrive at an active state. The AWS Console should show the following status when the cluster is ready.

[Elasticsearch Dashboard picture]

You could also check this via AWS CLI:
```
aws es describe-elasticsearch-domain --domain-name kubernetes-logs --query 'DomainStatus.Processing'
```
If the output value is false that means the domain has been processed and is now available to use.

Feel free to move on to the next section for now.

# Deploy Fluentd
```
mkdir ~/environment/fluentd
cd ~/environment/fluentd
wget https://eksworkshop.com/logging/deploy.files/fluentd.yml
```
Explore the fluentd.yml to see what is being deployed. There is a link at the bottom of this page. The Fluentd log agent configuration is located in the Kubernetes ConfigMap. Fluentd will be deployed as a DaemonSet, i.e. one pod per worker node. In our case, a 3 node cluster is used and so 3 pods will be shown in the output when we deploy.

Update REGION and CLUSTER_NAME environment variables in fluentd.yml as required. They are set to us-west-2 and eksworkshop-eksctl by default.
```
kubectl apply -f ~/environment/fluentd/fluentd.yml
```
Watch for all of the pods to change to running status
```
kubectl get pods -w --namespace=kube-system
```
We are now ready to check that logs are arriving in CloudWatch Logs

Select the region that is mentioned in fluentd.yml to browse the Cloudwatch Log Group if required.
Related files
fluentd.yml (5 ko) 

# Configure CloudWatch Logs and Kibana
## Configure CloudWatch Logs Subscription

CloudWatch Logs can be delivered to other services such as Amazon Elasticsearch for custom processing. This can be achieved by subscribing to a real-time feed of log events. A subscription filter defines the filter pattern to use for filtering which log events gets delivered to Elasticsearch, as well as information about where to send matching log events to.

In this section, we’ll subscribe to the CloudWatch log events from the fluent-cloudwatch stream from the eks/eksworkshop-eksctl log group. This feed will be streamed to the Elasticsearch cluster.

Original instructions for this are available at:

http://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_ES_Stream.html

    Workshop in your own account
```
cat <<EoF > ~/environment/iam_policy/lambda.json
{
   "Version": "2012-10-17",
   "Statement": [
   {
     "Effect": "Allow",
     "Principal": {
        "Service": "lambda.amazonaws.com"
     },
   "Action": "sts:AssumeRole"
   }
 ]
}
EoF

aws iam create-role --role-name lambda_basic_execution --assume-role-policy-document file://~/environment/iam_policy/lambda.json

aws iam attach-role-policy --role-name lambda_basic_execution --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

Go to the CloudWatch Logs console

Select the log group /eks/eksworkshop-eksctl/containers. Click on Actions and select Stream to Amazon ElasticSearch Service. Stream to ElasticSearch

Select the ElasticSearch Cluster kubernetes-logs and IAM role lambda_basic_execution

Subscribing to logs

Click Next

Select Common Log Format and click Next

ES Log Format

Review the configuration. Click Next and then Start Streaming

Review ES Subscription

Cloudwatch page is refreshed to show that the filter was successfully created
## Configure Kibana

In Amazon Elasticsearch console, select the kubernetes-logs under My domains

ElasticSearch Details

Open the Kibana dashboard from the link. After a few minutes, records will begin to be indexed by ElasticSearch. You’ll need to configure an index patterns in Kibana.

Set Index Pattern as cwl-* and click Next

Index Pattern

Select @timestamp from the dropdown list and select Create index pattern

Index Pattern

Kibana Summary

Click on Discover and explore your logs

Kibana Dashboard

# Cleanup Logging
```
cd ~/environment
kubectl delete -f ~/environment/fluentd/fluentd.yml
rm -rf ~/environment/fluentd/
aws es delete-elasticsearch-domain --domain-name kubernetes-logs
aws logs delete-log-group --log-group-name /eks/eksworkshop-eksctl/containers
rm -rf ~/environment/iam_policy/
```