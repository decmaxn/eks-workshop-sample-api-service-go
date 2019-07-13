# Launch using Terraform

We also have some very powerful partner tools that allow us to automate much of the experience of creating an EKS cluster, simplifying the process.

In this module, we will highlight a community created Terraform module, and use it to launch and configure our EKS cluster and nodes.

## Prerequisites

For this chapter, we need to download the Terraform binary:
```
curl -kLo /tmp/terraform.zip "https://releases.hashicorp.com/terraform/0.11.7/terraform_0.11.7_linux_amd64.zip"
sudo unzip -d /usr/local/bin/ /tmp/terraform.zip
sudo chmod +x /usr/local/bin/terraform
terraform version
```
## Get Terraform Module

Now let’s clone the community terraform module for EKS:
```
cd ${HOME}/environment/
git clone https://github.com/WesleyCharlesBlake/terraform-aws-eks.git
cd terraform-aws-eks
```
## Launch EKS Cluster

We start by initializing the Terraform state:
```
terraform init
```
We can now plan our deployment:
```
terraform plan -var 'cluster-name=eksworkshop-tf' -var 'desired-capacity=3' -out eksworkshop-tf
terraform apply "eksworkshop-tf"
```
Applying the fresh terraform plan will take approximately 15 minutes

## Create Kubeconfig File

Now that we have the cluster running, we need to create the KubeConfig file that will be used to manage the cluster.

The terraform module stores the kubeconfig information in it’s state store. We can view it with this command:
```
terraform output kubeconfig
```
And we can save it for use with this command:
```
terraform output kubeconfig > ${HOME}/.kube/config-eksworkshop-tf
```
We now need to add this new config to the KubeCtl Config list:
```
export KUBECONFIG=${HOME}/.kube/config-eksworkshop-tf:${HOME}/.kube/config
echo "export KUBECONFIG=${KUBECONFIG}" >> ${HOME}/.bashrc
```
## Create the Worker ConfigMap

The terraform state also contains a config-map we can use for our EKS workers.

View the configmap:
```
terraform output config-map
```
Save the config-map:
```
terraform output config-map > /tmp/config-map-aws-auth.yml
```
Apply the config-map:
```
kubectl apply -f /tmp/config-map-aws-auth.yml
```
Confirm your Nodes:
```
kubectl get nodes
```
Congratulations!

You now have a fully working Amazon EKS Cluster that is ready to use!

## Cleanup

In order to delete the resources created for this EKS cluster, run the following commands:

View the plan:
```
terraform plan -destroy -out eksworkshop-destroy-tf
```
Execute the plan:
```
terraform apply "eksworkshop-destroy-tf"
```
Destroying all the resources will take approximately 15 minutes
