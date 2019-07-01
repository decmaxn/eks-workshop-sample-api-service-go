# Batch Processing with Argo
## Batch Processing

In this Chapter, we will deploy common batch processing scenarios using Kubernetes and Argo.

Argo Logo

What is Argo?

Argo is an open source container-native workflow engine for getting work done on Kubernetes. Argo is implemented as a Kubernetes CRD (Custom Resource Definition).

    Define workflows where each step in the workflow is a container.
    Model multi-step workflows as a sequence of tasks or capture the dependencies between tasks using a graph (DAG).
    Easily run compute intensive jobs for machine learning or data processing in a fraction of the time using Argo workflows on Kubernetes.

## Introduction
### Introduction

Batch processing refers to performing units of work, referred to as a job in a repetitive and unattended fashion. Jobs are typically grouped together and processed in batches (hence the name).

Kubernetes includes native support for running Jobs. Jobs can run multiple pods in parallel until receiving a set number of completions. Each pod can contain multiple containers as a single unit of work.

Argo enhances the batch processing experience by introducing a number of features:

    Steps based declaration of workflows
    Artifact support
    Step level inputs & outputs
    Loops
    Conditionals
    Visualization (using Argo Dashboard)
    …and more

In this module, we will build a simple Kubernetes Job, recreate that job in Argo, and add common features and workflows for more advanced batch processing.

## Kubernetes Jobs
### Kubernetes Jobs

A job creates one or more pods and ensures that a specified number of them successfully terminate. As pods successfully complete, the job tracks the successful completions. When a specified number of successful completions is reached, the job itself is complete. Deleting a Job will cleanup the pods it created.

Save the below manifest as ‘job-whalesay.yaml’ using your favorite editor.
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ cat<<EOF > job-whalesay.yaml
> apiVersion: batch/v1
> kind: Job
> metadata:
>   name: whalesay
> spec:
>   template:
>     spec:
>       containers:
>       - name: whalesay
>         image: docker/whalesay
>         command: ["cowsay",  "This is a Kubernetes Job!"]
>       restartPolicy: Never
>   backoffLimit: 4
> EOF
```

Run a sample Kubernetes Job using the whalesay image.
```
kubectl apply -f job-whalesay.yaml
```
Wait until the job has completed successfully.
```
kubectl get job/whalesay

NAME       DESIRED   SUCCESSFUL   AGE
whalesay   1         1            2m
```
Confirm the output.
```
kubectl logs -l job-name=whalesay

 ___________________________ 
< This is a Kubernetes Job! >
 --------------------------- 
    \
     \
      \     
                    ##        .            
              ## ## ##       ==            
           ## ## ## ##      ===            
       /""""""""""""""""___/ ===        
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
       \______ o          __/            
        \    \        __/             
          \____\______/ 
          
```

# Install Argo CLI
Install Argo CLI

Before we can get started configuring argo we’ll need to first install the command line tools that you will interact with. To do this run the following.
```
sudo curl -sSL -o /usr/local/bin/argo https://github.com/argoproj/argo/releases/download/v2.2.1/argo-linux-amd64
sudo chmod +x /usr/local/bin/argo
```

# Deploy Argo

Argo run in its own namespace and deploys as a CustomResourceDefinition.

Deploy the Controller and UI.
```
kubectl create ns argo
kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo/v2.2.1/manifests/install.yaml
```
```
namespace/argo created
customresourcedefinition.apiextensions.k8s.io/workflows.argoproj.io created
serviceaccount/argo created
serviceaccount/argo-ui created
clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-admin created
clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-edit created
clusterrole.rbac.authorization.k8s.io/argo-aggregate-to-view created
clusterrole.rbac.authorization.k8s.io/argo-cluster-role created
clusterrole.rbac.authorization.k8s.io/argo-ui-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/argo-binding created
clusterrolebinding.rbac.authorization.k8s.io/argo-ui-binding created
configmap/workflow-controller-configmap created
service/argo-ui created
deployment.apps/argo-ui created
deployment.apps/workflow-controller created
```
To use advanced features of Argo for this demo, create a RoleBinding to grant admin privileges to the ‘default’ service account.

This is for demo purposes only. In any other environment, you should use Workflow RBAC to set appropriate permissions.
```
kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=default:default
```

# Configure Artifact Repository
## Configure Artifact Repository

Argo uses an artifact repository to pass data between jobs in a workflow, known as artifacts. Amazon S3 can be used as an artifact repository.

Let’s create a S3 bucket using the AWS CLI.
    Workshop in your own account
```
aws s3 mb s3://batch-artifact-repository-${ACCOUNT_ID}/
```
Next, edit the workflow-controller ConfigMap to use the S3 bucket.
```
kubectl edit -n argo configmap/workflow-controller-configmap
```
Add the following lines to the end of the ConfigMap, substituting your Account ID for {{ACCOUNT_ID}}:
```
data:
  config: |
    artifactRepository:
      s3:
        bucket: batch-artifact-repository-{{ACCOUNT_ID}}
        endpoint: s3.amazonaws.com
```
## Create an IAM Policy

In order for Argo to read from/write to the S3 bucket, we need to configure an inline policy and add it to the EC2 instance profile of the worker nodes.

First, we will need to ensure the Role Name our workers use is set in our environment:
```
test -n "$ROLE_NAME" && echo ROLE_NAME is "$ROLE_NAME" || echo ROLE_NAME is not set
```
If you receive an error or empty response, expand the steps below to export.
Expand here if you need to export the Role Name
```
# Example Output
ROLE_NAME is eks-workshop-nodegroup
```
Create and policy and attach to the worker node role.

    Workshop in your own account

```
mkdir ~/environment/batch_policy
cat <<EoF > ~/environment/batch_policy/k8s-s3-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*"
      ],
      "Resource": [
        "arn:aws:s3:::batch-artifact-repository-${ACCOUNT_ID}",
        "arn:aws:s3:::batch-artifact-repository-${ACCOUNT_ID}/*"
      ]
    }
  ]
}
EoF

aws iam put-role-policy --role-name $ROLE_NAME --policy-name S3-Policy-For-Worker --policy-document file://~/environment/batch_policy/k8s-s3-policy.json
```
Validate that the policy is attached to the role
```
aws iam get-role-policy --role-name $ROLE_NAME --policy-name S3-Policy-For-Worker
```

# Simple Batch Workflow
Simple Batch Workflow

Save the below manifest as ‘workflow-whalesay.yaml’ using your favorite editor and let’s deploy the whalesay example from before using Argo.
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ cat<<EoF > workflow-whalesay.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Workflow
> metadata:
>   generateName: whalesay-
> spec:
>   entrypoint: whalesay
>   templates:
>   - name: whalesay
>     container:
>       image: docker/whalesay
>       command: [cowsay]
>       args: ["This is an Argo Workflow!"]
> EoF
```
Now deploy the workflow using the argo CLI.

You can also run workflow specs directly using kubectl but the argo CLI provides syntax checking, nicer output, and requires less typing. For the equivalent kubectl commands, see Argo CLI.
```
argo submit --watch workflow-whalesay.yaml
```
```
Name:                whalesay-2kfxb
Namespace:           default
ServiceAccount:      default
Status:              Succeeded
Created:             Sat Nov 17 10:32:13 -0500 (3 seconds ago)
Started:             Sat Nov 17 10:32:13 -0500 (3 seconds ago)
Finished:            Sat Nov 17 10:32:16 -0500 (now)
Duration:            3 seconds

STEP               PODNAME         DURATION  MESSAGE
 ✔ whalesay-2kfxb  whalesay-2kfxb  2s        
```
Make a note of the workflow’s name from your output (It should be similar to whalesay-xxxxx).

Confirm the output by running the following command, substituting name of your workflow for “whalesay-xxxxx”:
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ argo logs whalesay-rcdz5
 ___________________________ 
< This is an Argo Workflow! >
 --------------------------- 
    \
     \
      \     
                    ##        .            
              ## ## ##       ==            
           ## ## ## ##      ===            
       /""""""""""""""""___/ ===        
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
       \______ o          __/            
        \    \        __/             
          \____\______/     
```

# Advanced Batch Workflow
Advanced Batch Workflow

Let’s take a look at a more complex workflow, involving passing artifacts between jobs, multiple dependencies, etc.

Save the below manifest as teardrop.yaml using your favorite editor.

apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: teardrop-
spec:
  entrypoint: teardrop
  templates:
  - name: create-chain
    container:
      image: alpine:latest
      command: ["sh", "-c"]
      args: ["touch /tmp/message"]
    outputs:
      artifacts:
      - name: chain
        path: /tmp/message
  - name: whalesay
    inputs:
      parameters:
      - name: message
      artifacts:
      - name: chain
        path: /tmp/message
    container:
      image: docker/whalesay
      command: ["sh", "-c"]
      args: ["echo Chain: ; cat /tmp/message* | sort | uniq | tee /tmp/message; cowsay This is Job {{inputs.parameters.message}}! ; echo {{inputs.parameters.message}} >> /tmp/message"]
    outputs:
      artifacts:
      - name: chain
        path: /tmp/message
  - name: whalesay-reduce
    inputs:
      parameters:
      - name: message
      artifacts:
      - name: chain-0
        path: /tmp/message.0
      - name: chain-1
        path: /tmp/message.1
    container:
      image: docker/whalesay
      command: ["sh", "-c"]
      args: ["echo Chain: ; cat /tmp/message* | sort | uniq | tee /tmp/message; cowsay This is Job {{inputs.parameters.message}}! ; echo {{inputs.parameters.message}} >> /tmp/message"]
    outputs:
      artifacts:
      - name: chain
        path: /tmp/message
  - name: teardrop
    dag:
      tasks:
      - name: create-chain
        template: create-chain
      - name: Alpha
        dependencies: [create-chain]
        template: whalesay
        arguments:
          parameters: [{name: message, value: Alpha}]
          artifacts:
            - name: chain
              from: "{{tasks.create-chain.outputs.artifacts.chain}}"
      - name: Bravo
        dependencies: [Alpha]
        template: whalesay
        arguments:
          parameters: [{name: message, value: Bravo}]
          artifacts:
            - name: chain
              from: "{{tasks.Alpha.outputs.artifacts.chain}}"
      - name: Charlie
        dependencies: [Alpha]
        template: whalesay
        arguments:
          parameters: [{name: message, value: Charlie}]
          artifacts:
            - name: chain
              from: "{{tasks.Alpha.outputs.artifacts.chain}}"
      - name: Delta
        dependencies: [Bravo]
        template: whalesay
        arguments:
          parameters: [{name: message, value: Delta}]
          artifacts:
            - name: chain
              from: "{{tasks.Bravo.outputs.artifacts.chain}}"
      - name: Echo
        dependencies: [Bravo, Charlie]
        template: whalesay-reduce
        arguments:
          parameters: [{name: message, value: Echo}]
          artifacts:
            - name: chain-0
              from: "{{tasks.Bravo.outputs.artifacts.chain}}"
            - name: chain-1
              from: "{{tasks.Charlie.outputs.artifacts.chain}}"
      - name: Foxtrot
        dependencies: [Charlie]
        template: whalesay
        arguments:
          parameters: [{name: message, value: Foxtrot}]
          artifacts:
            - name: chain
              from: "{{tasks.create-chain.outputs.artifacts.chain}}"
      - name: Golf
        dependencies: [Delta, Echo]
        template: whalesay-reduce
        arguments:
          parameters: [{name: message, value: Golf}]
          artifacts:
            - name: chain-0
              from: "{{tasks.Delta.outputs.artifacts.chain}}"
            - name: chain-1
              from: "{{tasks.Echo.outputs.artifacts.chain}}"
      - name: Hotel
        dependencies: [Echo, Foxtrot]
        template: whalesay-reduce
        arguments:
          parameters: [{name: message, value: Hotel}]
          artifacts:
            - name: chain-0
              from: "{{tasks.Echo.outputs.artifacts.chain}}"
            - name: chain-1
              from: "{{tasks.Foxtrot.outputs.artifacts.chain}}"

This workflow uses a Directed Acyclic Graph (DAG) to explicitly define job dependencies. Each job in the workflow calls a whalesay template and passes a parameter with a unique name. Some jobs call a whalesay-reduce template which accepts multiple artifacts and combines them into a single artifact.

Each job in the workflow pulls the artifact(s) and lists them in the “Chain”, then calls whalesay for the current job. Each job will then have a list of the previous job dependency chain (list of all jobs that had to complete before current job could run).

Run the workflow.
```
argo submit --watch teardrop.yaml
```
```
Name:                teardrop-jfg5w
Namespace:           default
ServiceAccount:      default
Status:              Succeeded
Created:             Sat Nov 17 16:01:42 -0500 (7 minutes ago)
Started:             Sat Nov 17 16:01:42 -0500 (7 minutes ago)
Finished:            Sat Nov 17 16:03:35 -0500 (5 minutes ago)
Duration:            1 minute 53 seconds

STEP               PODNAME                    DURATION  MESSAGE
 ✔ teardrop-jfg5w                                       
 ├-✔ create-chain  teardrop-jfg5w-3938249022  3s        
 ├-✔ Alpha         teardrop-jfg5w-3385521262  6s        
 ├-✔ Bravo         teardrop-jfg5w-1878939134  35s       
 ├-✔ Charlie       teardrop-jfg5w-3753534620  35s       
 ├-✔ Foxtrot       teardrop-jfg5w-2036090354  5s        
 ├-✔ Delta         teardrop-jfg5w-37094256    34s       
 ├-✔ Echo          teardrop-jfg5w-4165010455  31s       
 ├-✔ Hotel         teardrop-jfg5w-2342859904  4s        
 └-✔ Golf          teardrop-jfg5w-1687601882  30s       
```
Continue to the Argo Dashboard to explore this model further.

# Argo Dashboard
## Argo Dashboard

[Argo Dashboard picture]

Argo UI lists the workflows and visualizes each workflow (very handy for our last workflow).

To connect, use the same proxy connection setup in Deploy the Official Kubernetes Dashboard.
Show me the command
```
kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &
```
This will start the proxy, listen on port 8080, listen on all interfaces, and will disable the filtering of non-localhost requests.

This command will continue to run in the background of the current terminal’s session.

We are disabling request filtering, a security feature that guards against XSRF attacks. This isn’t recommended for a production environment, but is useful for our dev environment.

To access the Argo Dashboard:

    In your Cloud9 environment, click Preview / Preview Running Application
    Scroll to the end of the URL and append:
```
/api/v1/namespaces/argo/services/argo-ui/proxy/
```
You will see the teardrop workflow from Advanced Batch Workflow. Click on it to see a visualization of the workflow.

Argo Workflow

The workflow should relatively look like a teardrop, and provide a live status for each job. Click on Hotel to see a summary of the Hotel job.

Argo Hotel Job

This details basic information about the job, and includes a link to the Logs. The Hotel job logs list the job dependency chain and the current whalesay, and should look similar to:
```
Chain:
Alpha
Bravo
Charlie
Echo
Foxtrot
____________________
< This is Job Hotel! >
--------------------
   \
    \
     \
                   ##        .
             ## ## ##       ==
          ## ## ## ##      ===
      /""""""""""""""""___/ ===
 ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
      \______ o          __/
       \    \        __/
         \____\______/
```
Explore the other jobs in the workflow to see each job’s status and logs.

# Cleanup
## Cleanup
Delete all workflows
```
argo delete --all
```
Remove Artifact Repository Bucket

    Workshop at AWS event
    Workshop in your own account

```
aws s3 rb s3://batch-artifact-repository-${ACCOUNT_ID}/ --force
```
### Undeploy Argo
```
kubectl delete -n argo -f https://raw.githubusercontent.com/argoproj/argo/v2.2.1/manifests/install.yaml
kubectl delete ns argo
```
Cleanup Kubernetes Job
```
kubectl delete job/whalesay
```
