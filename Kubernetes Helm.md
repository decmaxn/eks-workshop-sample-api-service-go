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

