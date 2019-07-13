# eks-workshop-sample-api-service-go

A sample Kubernetes service used in the [EKS Workshop](https://eksworkshop.com/) CI/CD Pipeline module.

The Dockerfile is a [multi-stage](https://docs.docker.com/develop/develop-images/multistage-build/) build that
compiles the Go application and then packages it in a minimal image that pulls from [scratch](https://hub.docker.com/_/scratch/).
The size of this Docker image is ~ 3.2 MiB.

The buildspec.yml file is used by the [AWS CodeBuild](https://aws.amazon.com/codebuild/) stage. In this file, it pulls down
kubectl, builds the container image, pushes the image to [Amazon ECR](https://aws.amazon.com/ecr/) and then deploys the change to the
[Amazon EKS Cluster](https://aws.amazon.com/eks/).

In the hello-k8s.yml file, you will find the Kubernetes [service](https://kubernetes.io/docs/concepts/services-networking/service/) and
[deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) definitions. The service is configured with
a [LoadBalancer](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/) which prompts Kubernetes
to launch an external load balancer using an [AWS ELB](https://aws.amazon.com/elasticloadbalancing/).

# Another eksworkshop
Accidentally find with introduction.html following the eksworkshop website, like https://eksworkshop.com/introduction.html, actually lead to a difficult workshop. 
```
    Built EKS using a partner provided tool called eksctl
    Built EKS using a community contributed terraform module
    Built EKS using the recommended steps from AWS
    Deployed the Kubernetes Dashboard
```
The difference is it introduces how to use terraform or Cloudformation to create EKS cluster instead reply on eksctl. Maybe this is an older version, but definitly helpful to understand the cluster. 

Refer to Terraform.md and Cloudformation.md for details.