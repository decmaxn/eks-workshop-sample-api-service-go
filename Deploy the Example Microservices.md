# Deploy NodeJS Backend API
Let’s bring up the NodeJS Backend API!

Copy/Paste the following commands into your Cloud9 workspace:
```
cd ~/ecsdemo-nodejs
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```
We can watch the progress by looking at the deployment status:
```
kubectl get deployment ecsdemo-nodejs
```
# Deploy Crystal Backend API
Let’s bring up the Crystal Backend API!

Copy/Paste the following commands into your Cloud9 workspace:
```
cd ~/ecsdemo-crystal
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```
We can watch the progress by looking at the deployment status:
```
kubectl get deployment ecsdemo-crystal
```

# Let's check Service Types
Before we bring up the frontend service, let’s take a look at the service types we are using: This is kubernetes/service.yaml for our frontend service:
```
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-frontend
spec:
  selector:
    app: ecsdemo-frontend
  type: LoadBalancer
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
```
Notice type: LoadBalancer: This will configure an ELB to handle incoming traffic to this service.

Compare this to kubernetes/service.yaml for one of our backend services:
```
apiVersion: v1
kind: Service
metadata:
  name: ecsdemo-nodejs
spec:
  selector:
    app: ecsdemo-nodejs
  ports:
   -  protocol: TCP
      port: 80
      targetPort: 3000
```
Notice there is no specific service type described. When we check the kubernetes documentation we find that the default type is ClusterIP. This Exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster.
# Ensure the ELB Service Role exists
In AWS accounts that have never created a load balancer before, it’s possible that the service role for ELB might not exist yet.

We can check for the role, and create it if it’s missing.

Copy/Paste the following commands into your Cloud9 workspace:
```
aws iam get-role --profile vma --role-name "AWSServiceRoleForElasticLoadBalancing" || aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"
```
# Deploy Frontend Service

Let’s bring up the Ruby Frontend!

Copy/Paste the following commands into your Cloud9 workspace:
```
cd ~/ecsdemo-frontend
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```
We can watch the progress by looking at the deployment status:
```
kubectl get deployment ecsdemo-frontend
```
# Find the Service Address

Now that we have a running service that is type: LoadBalancer we need to find the ELB’s address. We can do this by using the get services operation of kubectl:
```
kubectl get service ecsdemo-frontend
```
Notice the field isn’t wide enough to show the FQDN of the ELB. We can adjust the output format with this command:
```
kubectl get service ecsdemo-frontend -o wide
```
If we wanted to use the data programatically, we can also output via json. This is an example of how we might be able to make use of json output:
```
ELB=$(kubectl get service ecsdemo-frontend -o json | jq -r '.status.loadBalancer.ingress[].hostname')
curl -m3 -v $ELB
```
It will take several minutes for the ELB to become healthy and start passing traffic to the frontend pods.

You should also be able to copy/paste the loadBalancer hostname into your browser and see the application running. Keep this tab open while we scale the services up on the next page.

# Scale the Backend Services

When we launched our services, we only launched one container of each. We can confirm this by viewing the running pods:
```
kubectl get deployments
```
Now let’s scale up the backend services:
```
kubectl scale deployment ecsdemo-nodejs --replicas=3
kubectl scale deployment ecsdemo-crystal --replicas=3
```
Confirm by looking at deployments again:
```
kubectl get deployments
```
Also, check the browser tab where we can see our application running. You should now see traffic flowing to multiple backend services.

# Scale the Frontend
Let’s also scale our frontend service the same way:
```
kubectl get deployments
kubectl scale deployment ecsdemo-frontend --replicas=3
kubectl get deployments
```
Check the browser tab where we can see our application running. You should now see traffic flowing to multiple frontend services.
# Cleanup the applications

To delete the resources created by the applications, we should delete the application deployments:

Undeploy the applications:
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