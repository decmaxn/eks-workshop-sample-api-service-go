# Deploy Jenkins

In this Chapter, we will deploy Jenkins using the helm package manager we installed in the last module.
```
[ec2-user@bastion eks-workshop-sample-api-service-go]$ cat <<EoF > ~/environment/rbac.yaml
> ---
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: tiller
>   namespace: kube-system
> ---
> apiVersion: rbac.authorization.k8s.io/v1beta1
> kind: ClusterRoleBinding
> metadata:
>   name: tiller
> roleRef:
>   apiGroup: rbac.authorization.k8s.io
>   kind: ClusterRole
>   name: cluster-admin
> subjects:
>   - kind: ServiceAccount
>     name: tiller
>     namespace: kube-system
> EoF
[ec2-user@bastion eks-workshop-sample-api-service-go]$ kubectl apply -f ~/environment/rbac.yaml
serviceaccount/tiller created
```

With our Storage Class configured we then need to create our jenkins setup. To do this we’ll just use the helm cli with a couple flags.

In a production system you should be using a values.yaml file so that you can manage the drift as you need to update releases
## Install Jenkins
```
$ helm install stable/jenkins --set rbac.create=true --name cicd
NAME:   cicd
LAST DEPLOYED: Mon Jul  1 19:00:39 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                DATA  AGE
cicd-jenkins        5     0s
cicd-jenkins-tests  1     0s

==> v1/Deployment
NAME          READY  UP-TO-DATE  AVAILABLE  AGE
cicd-jenkins  0/1    1           0          0s

==> v1/PersistentVolumeClaim
NAME          STATUS   VOLUME  CAPACITY  ACCESS MODES  STORAGECLASS  AGE
cicd-jenkins  Pending  gp2     0s

==> v1/Pod(related)
NAME                           READY  STATUS   RESTARTS  AGE
cicd-jenkins-6796946b64-cwxt7  0/1    Pending  0         0s

==> v1/Role
NAME                          AGE
cicd-jenkins-schedule-agents  0s

==> v1/RoleBinding
NAME                          AGE
cicd-jenkins-schedule-agents  0s

==> v1/Secret
NAME          TYPE    DATA  AGE
cicd-jenkins  Opaque  2     0s

==> v1/Service
NAME                TYPE          CLUSTER-IP      EXTERNAL-IP  PORT(S)         AGE
cicd-jenkins        LoadBalancer  10.100.115.180  <pending>    8080:31832/TCP  0s
cicd-jenkins-agent  ClusterIP     10.100.176.226  <none>       50000/TCP       0s

==> v1/ServiceAccount
NAME          SECRETS  AGE
cicd-jenkins  1        0s


NOTES:
1. Get your 'admin' user password by running:
  printf $(kubectl get secret --namespace default cicd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        You can watch the status of by running 'kubectl get svc --namespace default -w cicd-jenkins'
  export SERVICE_IP=$(kubectl get svc --namespace default cicd-jenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
  echo http://$SERVICE_IP:8080/login

3. Login with the password from step 1 and the username: admin


For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine
```
The output of this command will give you some additional information such as the admin password and the way to get the host name of the ELB that was provisioned.

Let’s give this some time to provision and while we do let’s watch for pods to boot.

kubectl get pods -w

You should see the pods in init, pending or running state.

Follow the NOTES section and...

## Logging In

Now that we have the ELB address of your jenkins instance we can go an navigate to that address in another window.

## Cleanup

To delete Jenkins, run:
```
helm del --purge cicd
```