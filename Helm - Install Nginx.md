# Kubernetes Helm

Helm is a package manager for Kubernetes that packages multiple Kubernetes resources into a single logical deployment unit called Chart.

In this chapter, we’ll cover installing Helm. Once installed, we’ll demonstrate how Helm can be used to deploy a simple NGINX webserver, and a more sophisticated microservice.


## Install Helm on EKS

Helm is a package manager and application management tool for Kubernetes that packages multiple Kubernetes resources into a single logical deployment unit called Chart.

Helm helps you to:

    Achieve a simple (one command) and repeatable deployment
    Manage application dependency, using specific versions of other application and services
    Manage multiple deployment configurations: test, staging, production and others
    Execute post/pre deployment jobs during application deployment
    Update/rollback and test application deployments

### Install Helm CLI
Before we can get started configuring helm we’ll need to first install the command line tools that you will interact with. To do this run the following.
```
cd ~
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod +x get_helm.sh
./get_helm.sh
```
Once you install helm, the command will prompt you to run ‘helm init’. Do not run ‘helm init’. Follow the instructions to configure helm using Kubernetes RBAC and then install tiller as specified below If you accidentally run ‘helm init’, you can safely uninstall tiller by running ‘helm reset –force’
#### Configure Helm access with RBAC

Helm relies on a service called tiller that requires special permission on the kubernetes cluster, so we need to build a Service Account for tiller to use. We’ll then apply this to the cluster.

To create a new service account manifest:
```
cat <<EoF > rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EoF
```
Next apply the config:
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ kubectl apply -f rbac.yaml
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created
```
Then we can install tiller using the helm tooling
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ helm init --service-account tiller
Creating /home/ec2-user/.helm 
Creating /home/ec2-user/.helm/repository 
Creating /home/ec2-user/.helm/repository/cache 
Creating /home/ec2-user/.helm/repository/local 
Creating /home/ec2-user/.helm/plugins 
Creating /home/ec2-user/.helm/starters 
Creating /home/ec2-user/.helm/cache/archive 
Creating /home/ec2-user/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /home/ec2-user/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
```
This will install tiller into the cluster which gives it access to manage resources in your cluster.
## Deploy Nginx With Helm

In this Chapter, we will dig deeper with Helm and demonstrate how to install the NGINX web server via the following steps:

### Update the Chart Repository
Helm uses a packaging format called Charts. A Chart is a collection of files that describe k8s resources.

Charts can be simple, describing something like a standalone web server (which is what we are going to create), but they can also be more complex, for example, a chart that represents a full web application stack included web servers, databases, proxies, etc.

Instead of installing k8s resources manually via kubectl, we can use Helm to install pre-defined Charts faster, with less chance of typos or other operator errors.

When you install Helm, you are provided with a default repository of Charts from the official Helm Chart Repository.

This is a very dynamic list that always changes due to updates and new additions. To keep Helm’s local list updated with all these changes, we need to occasionally run the repository update command.

To update Helm’s local list of Charts, run: And you should see something similar to:
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
```
Next, we’ll search for the NGINX web server Chart.
### Search the Chart Repository
Now that our repository Chart list has been updated, we can search for Charts.

To list all Charts:
```
helm search
```
That should output something similiar to:
```
NAME                                    CHART VERSION   APP VERSION                     DESCRIPTION                                                 
stable/acs-engine-autoscaler            2.2.0           2.1.1                           Scales worker...
stable/aerospike                        0.1.7           v3.14.1.2                       A Helm chart...
...
```
You can see from the output that it dumped the list of all Charts it knows about. In some cases that may be useful, but an even more useful search would involve a keyword argument. So next, we’ll search just for NGINX:
```
helm search nginx
```
That results in:
```
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                                 
stable/nginx-ingress            0.31.0          0.20.0          An nginx Ingress ...
stable/nginx-ldapauth-proxy     0.1.2           1.13.5          nginx proxy ...
stable/nginx-lego               0.3.1                           Chart for...
stable/gcloud-endpoints         0.1.2           1               DEPRECATED Develop...
...
```
This new list of Charts are specific to nginx, because we passed the nginx argument to the search command.
### Add the Bitnami Repository
In the last slide, we saw that NGINX offers many different products via the default Helm Chart repository, but the NGINX standalone web server is not one of them.

After a quick web search, we discover that there is a Chart for the NGINX standalone web server available via the Bitnami Chart repository.

To add the Bitnami Chart repo to our local list of searchable charts:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```
Once that completes, we can search all Bitnami Charts:
```
helm search bitnami
```
Which results in:
```
NAME                                    CHART VERSION   APP VERSION             DESCRIPTION                                                 
bitnami/bitnami-common                  0.0.3           0.0.1                   Chart with...        
bitnami/apache                          2.1.2           2.4.37                  Chart for Apache...                              
bitnami/cassandra                       0.1.0           3.11.3                  Apache Cassandra...
...
```
Search once again for NGINX:
```
helm search nginx
```
Now we are seeing more NGINX options, across both repositories:
```
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION                                                 
bitnami/nginx                           1.1.2           1.14.1          Chart for the nginx server                                  
bitnami/nginx-ingress-controller        2.1.4           0.20.0          Chart for the nginx Ingress...                    
stable/nginx-ingress                    0.31.0          0.20.0          An nginx Ingress controller ...
```
Or even search the Bitnami repo, just for NGINX:
```
helm search bitnami/nginx
```
Which narrows it down to NGINX on Bitnami:
```
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION                           
bitnami/nginx                           1.1.2           1.14.1          Chart for the nginx server            
bitnami/nginx-ingress-controller        2.1.4           0.20.0          Chart for the nginx Ingress...
```
In both of those last two searches, we see
```
bitnami/nginx
```
as a search result. That’s the one we’re looking for, so let’s use Helm to install it to the EKS cluster.
### Install bitnami/nginx
Installing the Bitnami standalone NGINX web server Chart involves us using the helm install command.

When we install using Helm, we need to provide a deployment name, or a random one will be assigned to the deployment automatically.
Challenge:

How can you use Helm to deploy the bitnami/nginx chart?

HINT: Use the helm utility to install the bitnami/nginx chart and specify the name mywebserver for the Kubernetes deployment. Consult the helm install documentation or run the helm install --help command to figure out the syntax
Expand here to see the solution

Once you run this command, the output confirms the types of k8s objects that were created as a result:
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ helm install --name mywebserver bitnami/nginx
NAME:   mywebserver
LAST DEPLOYED: Fri Jun 28 05:04:41 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                                READY  STATUS             RESTARTS  AGE
mywebserver-nginx-686d48cfc4-8cbh9  0/1    ContainerCreating  0         0s

==> v1/Service
NAME               TYPE          CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
mywebserver-nginx  LoadBalancer  10.100.98.115  <pending>    80:32405/TCP  0s

==> v1beta1/Deployment
NAME               READY  UP-TO-DATE  AVAILABLE  AGE
mywebserver-nginx  0/1    1           0          0s


NOTES:
```
In the following kubectl command examples, it may take a minute or two for each of these objects’ DESIRED and CURRENT values to match; if they don’t match on the first try, wait a few seconds, and run the command again to check the status.

The first object shown in this output is a Deployment. A Deployment object manages rollouts (and rollbacks) of different versions of an application.

You can inspect this Deployment object in more detail by running the following command:
```
kubectl describe deployment mywebserver-nginx
```
The next object shown created by the Chart is a Pod. A Pod is a group of one or more containers.

To verify the Pod object was successfully deployed, we can run the following command:
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ kubectl get pods -l app=mywebserver-nginx
NAME                                 READY   STATUS    RESTARTS   AGE
mywebserver-nginx-686d48cfc4-8cbh9   1/1     Running   0          2m30s
```
The third object that this Chart creates for us is a Service The Service enables us to contact this NGINX web server from the Internet, via an Elastic Load Balancer (ELB).

To get the complete URL of this Service, run:
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ kubectl get service mywebserver-nginx -o wide
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)        AGE     SELECTOR
mywebserver-nginx   LoadBalancer   10.100.98.115   a3c6e26af996211e9bd4f06386a90bed-171473212.us-east-2.elb.amazonaws.com   80:32405/TCP   3m33s   app=mywebserver-nginx
```
Copy the value for EXTERNAL-IP, open a new tab in your web browser, and paste it in.

It may take a couple minutes for the ELB and its associated DNS name to become available; if you get an error, wait one minute, and hit reload.

When the Service does come online, you should see a welcome message similar to:

Congrats! You’ve now successfully deployed the NGINX standalone web server to your EKS cluster!
### Clean Up

To remove all the objects that the Helm Chart created, we can use Helm delete.

Before we delete it though, we can verify what we have running via the Helm list command:
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ helm list
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
mywebserver     1               Fri Jun 28 05:04:41 2019        DEPLOYED        nginx-3.3.2     1.16.0          default  
```
It was a lot of fun; we had some great times sending HTTP back and forth, but now its time to delete this deployment. To delete:
```
helm delete --purge mywebserver
```
kubectl will also demonstrate that our pods and service are no longer available:
```
kubectl get pods -l app=mywebserver-nginx
kubectl get service mywebserver-nginx -o wide
```
As would trying to access the service via the web browser via a page reload.

With that, cleanup is complete.