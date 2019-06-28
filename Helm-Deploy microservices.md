## Deploy Example Microservices Using Helm

In this chapter, we will demonstrate how to deploy microservices using a custom Helm Chart, instead of doing everything manually using kubectl.

For detailed information on working with chart templates, refer to the Helm docs
### Create a Chart

Helm charts have a structure similar to:
```
/eksdemo
  /Chart.yaml  # a description of the chart
  /values.yaml # defaults, may be overridden during install or upgrade
  /charts/ # May contain subcharts
  /templates/ # the template files themselves
  ...
```
We’ll follow this template, and create a new chart called eksdemo with the following commands:
```
cd ~/
helm create eksdemo
```

### Customize Defaults

If you look in the newly created eksdemo directory, you’ll see several files and directories. Specifically, inside the /templates directory, you’ll see:

    NOTES.txt: The “help text” for your chart. This will be displayed to your users when they run helm install.
    deployment.yaml: A basic manifest for creating a Kubernetes deployment
    service.yaml: A basic manifest for creating a service endpoint for your deployment
    _helpers.tpl: A place to put template helpers that you can re-use throughout the chart

We’re actually going to create our own files, so we’ll delete these boilerplate files
```
rm -rf ~/eksdemo/templates/
rm ~/eksdemo/Chart.yaml
rm ~/eksdemo/values.yaml
```
Create a new Chart.yaml file which will describe the chart

cat <<EoF > ~/eksdemo/Chart.yaml
apiVersion: v1
appVersion: "1.0"
description: A Helm chart for EKS Workshop Microservices application
name: eksdemo
version: 0.1.0
EoF

Next we’ll copy the manifest files for each of our microservices into the templates directory as servicename.yaml
```
#create subfolders for each template type
mkdir -p ~/eksdemo/templates/deployment
mkdir -p ~/eksdemo/templates/service

# Copy and rename frontend manifests
cp ~/ecsdemo-frontend/kubernetes/deployment.yaml ~/eksdemo/templates/deployment/frontend.yaml
cp ~/ecsdemo-frontend/kubernetes/service.yaml ~/eksdemo/templates/service/frontend.yaml

# Copy and rename crystal manifests
cp ~/ecsdemo-crystal/kubernetes/deployment.yaml ~/eksdemo/templates/deployment/crystal.yaml
cp ~/ecsdemo-crystal/kubernetes/service.yaml ~/eksdemo/templates/service/crystal.yaml

# Copy and rename nodejs manifests
cp ~/ecsdemo-nodejs/kubernetes/deployment.yaml ~/eksdemo/templates/deployment/nodejs.yaml
cp ~/ecsdemo-nodejs/kubernetes/service.yaml ~/eksdemo/templates/service/nodejs.yaml
```
All files in the templates directory are sent through the template engine. These are currently plain YAML files that would be sent to Kubernetes as-is.
#### Replace hard-coded values with template directives

Let’s replace some of the values with template directives to enable more customization by removing hard-coded values.

Open ~/eksdemo/templates/deployment/frontend.yaml in your Cloud9 editor.

The following steps should be completed seperately for frontend.yaml, crystal.yaml, and nodejs.yaml.

Under spec, find replicas: 1 and replace with the following:
```
replicas: {{ .Values.replicas }}
```
Under spec.template.spec.containers.image, replace the image with the correct template value from the table below:
```
Filename 	Value
frontend.yaml 	- image: {{ .Values.frontend.image }}:{{ .Values.version }}
crystal.yaml 	- image: {{ .Values.crystal.image }}:{{ .Values.version }}
nodejs.yaml 	- image: {{ .Values.nodejs.image }}:{{ .Values.version }}
```
Create a values.yaml file with our template defaults

This file will populate our template directives with default values.
```
cat <<EoF > ~/eksdemo/values.yaml
# Default values for eksdemo.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# Release-wide Values
replicas: 3
version: 'latest'

# Service Specific Values
nodejs:
  image: brentley/ecsdemo-nodejs
crystal:
  image: brentley/ecsdemo-crystal
frontend:
  image: brentley/ecsdemo-frontend
EoF
```
###Deploy the eksdemo Chart
Use the dry-run flag to test our templates

To test the syntax and validity of the Chart without actually deploying it, we’ll use the dry-run flag.

The following command will build and output the rendered templates without installing the Chart:
```
helm install --debug --dry-run --name workshop ~/eksdemo
```
Confirm that the values created by the template look correct.
Deploy the chart

Now that we have tested our template, lets install it.
```
helm install --name workshop ~/eksdemo
```
Similar to what we saw previously in the NGINX Helm Chart example, an output of the Deployment, Pod, and Service objects are output, similar to:
```
NAME:   workshop
LAST DEPLOYED: Fri Nov 16 21:42:00 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME              AGE
ecsdemo-crystal   0s
ecsdemo-frontend  0s
ecsdemo-nodejs    0s

==> v1/Deployment
ecsdemo-crystal   0s
ecsdemo-frontend  0s
ecsdemo-nodejs    0s

==> v1/Pod(related)

NAME                               READY  STATUS             RESTARTS  AGE
ecsdemo-crystal-764b9cb9bc-4dwqt   0/1    ContainerCreating  0         0s
ecsdemo-crystal-764b9cb9bc-hcb62   0/1    ContainerCreating  0         0s
ecsdemo-crystal-764b9cb9bc-vl7nr   0/1    ContainerCreating  0         0s
ecsdemo-frontend-67876457f6-2xrtb  0/1    ContainerCreating  0         0s
ecsdemo-frontend-67876457f6-bfnc5  0/1    ContainerCreating  0         0s
ecsdemo-frontend-67876457f6-rb6rg  0/1    ContainerCreating  0         0s
ecsdemo-nodejs-c458bf55d-994cq     0/1    ContainerCreating  0         0s
ecsdemo-nodejs-c458bf55d-9qtbm     0/1    ContainerCreating  0         0s
ecsdemo-nodejs-c458bf55d-s9zkh     0/1    ContainerCreating  0         0s
```

### Test the Service

To test the service our eksdemo Chart created, we’ll need to get the name of the ELB endpoint that was generated when we deployed the Chart:
```
kubectl get svc ecsdemo-frontend -o jsonpath="{.status.loadBalancer.ingress[*].hostname}"; echo
```
Copy that address, and paste it into a new tab in your browser. You should see something similar to:

### Rolling Back

Mistakes will happen during deployment, and when they do, Helm makes it easy to undo, or “roll back” to the previously deployed version.
#### Update the demo application chart with a breaking change

Open values.yaml and modify the image name under nodejs.image to brentley/ecsdemo-nodejs-non-existing. This image does not exist, so this will break our deployment.

Deploy the updated demo application chart:
```
helm upgrade workshop ~/eksdemo
```
The rolling upgrade will begin by creating a new nodejs pod with the new image. The new ecsdemo-nodejs Pod should fail to pull non-existing image. Run helm status command to see the ImagePullBackOff error:
```
helm status workshop
```
#### Rollback the failed upgrade

Now we are going to rollback the application to the previous working release revision.

First, list Helm release revisions:
```
helm history workshop
```
Then, rollback to the previous application revision (can rollback to any revision too):
```
# rollback to the 1st revision
helm rollback workshop 1
```
Validate workshop release status with:
```
helm status workshop
```
### Cleanup

To delete the workshop release, run:
```
helm del --purge workshop
```