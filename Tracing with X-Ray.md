# Tracing with X-Ray

As distributed systems evolve, monitoring and debugging services becomes challenging. Container-orchestration platforms like Kubernetes solve a lot of problems, but they also introduce new challenges for developers and operators in understanding how services interact and where latency exists. AWS X-Ray helps developers analyze and debug distributed services.

In this module, we are going to deploy the X-Ray agent as a DaemonSet, deploy sample front-end and back-end services that are instrumented with the X-Ray SDK, make some sample requests and then examine the traces and service maps in the AWS Management Console.


# Modify IAM Role

In order for the X-Ray daemon to communicate with the service, we need to add a policy to the worker nodes’ AWS Identity and Access Management (IAM) role.

First, we will need to ensure the Role Name our workers use is set in our environment:
```
test -n "$ROLE_NAME" && echo ROLE_NAME is "$ROLE_NAME" || echo ROLE_NAME is not set
```
Expand here if you need to export the Role Name
If ROLE_NAME is not set, please review: /eksctl/test/
```
# Example Output
ROLE_NAME is eks-workshop-nodegroup
````
    Workshop in your own account
```
aws iam attach-role-policy --role-name $ROLE_NAME \
--policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
```

## Deploy X-Ray DaemonSet

Now that we have modified the IAM role for the worker nodes to permit write operations to the X-Ray service, we are going to deploy the X-Ray DaemonSet to the EKS cluster. The X-Ray daemon will be deployed to each worker node in the EKS cluster. For reference, see the example implementation used in this module.

The AWS X-Ray SDKs are used to instrument your microservices. When using the DaemonSet in the example implementation, you need to configure it to point to xray-service.default:2000.

The following showcases how to configure the X-Ray SDK for Go. This is merely an example and not a required step in the workshop.
```
func init() {
	xray.Configure(xray.Config{
		DaemonAddr:     "xray-service.default:2000",
		LogLevel:       "info",
	})
}
```
To deploy the X-Ray DaemonSet:
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ kubectl create -f https://eksworkshop.com/x-ray/daemonset.files/xray-k8s-daemonset.yaml
serviceaccount/xray-daemon created
clusterrolebinding.rbac.authorization.k8s.io/xray-daemon created
daemonset.extensions/xray-daemon created
configmap/xray-config created
service/xray-service created
```
To see the status of the X-Ray DaemonSet:
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ kubectl describe daemonset xray-daemon
Name:           xray-daemon
Selector:       app=xray-daemon
Node-Selector:  <none>
Labels:         app=xray-daemon
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 3
Number of Nodes Misscheduled: 0
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
```
The folllowing is an example of the command output:

GitHub Edit

To view the logs for all of the X-Ray daemon pods run the following
```
kubectl logs -l app=xray-daemon
```

## Deploy Example Microservices

We now have the foundation in place to deploy microservices, which are instrumented with X-Ray SDKs, to the EKS cluster.

In this step, we are going to deploy example front-end and back-end microservices to the cluster. The example services are already instrumented using the X-Ray SDK for Go. Currently, X-Ray has SDKs for Go, Python, Node.js, Ruby, .NET and Java.
```
kubectl apply -f https://eksworkshop.com/x-ray/sample-front.files/x-ray-sample-front-k8s.yml

kubectl apply -f https://eksworkshop.com/x-ray/sample-back.files/x-ray-sample-back-k8s.yml
```
To review the status of the deployments, you can run:
```
kubectl describe deployments x-ray-sample-front-k8s x-ray-sample-back-k8s
```
For the status of the services, run the following command:
```
kubectl describe services x-ray-sample-front-k8s x-ray-sample-back-k8s
```
Once the front-end service is deployed, run the following command to get the Elastic Load Balancer (ELB) endpoint and open it in a browser.
```
kubectl get service x-ray-sample-front-k8s -o wide
```
After your ELB is deployed and available, open up the endpoint returned by the previous command in your browser and allow it to remain open. The front-end application makes a new request to the /api endpoint once per second, which in turn calls the back-end service. The JSON document displayed in the browser is the result of the request made to the back-end service.

This service was configured with a LoadBalancer so, an AWS Elastic Load Balancer (ELB) is launched by Kubernetes for the service. The EXTERNAL-IP column contains a value that ends with “elb.amazonaws.com” - the full value is the DNS address.

When the front-end service is first deployed, it can take up to several minutes for the ELB to be created and DNS updated.

## X-Ray Console

We now have the example microservices deployed, so we are going to investigate our Service Graph and Traces in X-Ray section of the AWS Management Console.

The Service map in the console provides a visual representation of the steps identified by X-Ray for a particular trace. Each resource that sends data to X-Ray within the same context appears as a service in the graph. In the example below, we can see that the x-ray-sample-front-k8s service is processing 39 transactions per minute with an average latency of 0.99ms per operation. Additionally, the x-ray-sample-back-k8s is showing an average latency of 0.08ms per transaction.

GitHub Edit

Next, go to the traces section in the AWS Management Console to view the execution times for the segments in the requests. At the top of the page, we can see the URL for the ELB endpoint and the corresponding traces below.

GitHub Edit

If you click on the link on the left in the Trace list section you will see the overall execution time for the request (0.5ms for the x-ray-sample-front-k8s which wraps other segments and subsegments), as well as a breakdown of the individual segments in the request. In this visualization, you can see the front-end and back-end segments and a subsegment named x-ray-sample-back-k8s-gen In the back-end service source code, we instrumented a subsegment that surrounds a random number generator.

In the Go example, the main segment is initialized in the xray.Handler helper, which in turn sets all necessary information in the http.Request context struct, so that it can be used when initializing the subsegment.

GitHub Edit

Click on the image to zoom

## Cleanup

Congratulations on completing the Tracing with X-Ray module.

The content for this module was based on the [Application Tracing on Kubernetes with AWS X-Ray](https://aws.amazon.com/blogs/compute/application-tracing-on-kubernetes-with-aws-x-ray/) blog post.

This module is not used in subsequent steps, so you can remove the resources now or at the end of the workshop.

Delete the Kubernetes example microservices deployed:
```
kubectl delete deployments x-ray-sample-front-k8s x-ray-sample-back-k8s

kubectl delete services x-ray-sample-front-k8s x-ray-sample-back-k8s
```
Delete the X-Ray DaemonSet:
```
kubectl delete -f https://eksworkshop.com/x-ray/daemonset.files/xray-k8s-daemonset.yaml
```
    Workshop in your own account

```
aws iam detach-role-policy --role-name $ROLE_NAME --policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
```