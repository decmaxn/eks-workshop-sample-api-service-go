By default, Kubernetes will restart a container if it crashes for any reason. It uses Liveness and Readiness probes which can be configured for running a robust application by identifying the healthy containers to send traffic to and restarting the ones when required.

In this section, we will understand how [liveness and readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes) are defined and test the same against different states of a pod. Below is the high level description of how these probes work.

Liveness probes are used in Kubernetes to know when a pod is alive or dead. A pod can be in a dead state for different reasons while Kubernetes kills and recreates the pod when liveness probe does not pass.

Readiness probes are used in Kubernetes to know when a pod is ready to serve traffic. Only when the readiness probe passes, a pod will receive traffic from the service. When readiness probe fails, traffic will not be sent to a pod until it passes.

The link above include this statement better explains the difference between liveness and readiness probes.
> Sometimes, applications are temporarily unable to serve traffic. For example, an application might need to load large data or configuration files during startup, or depend on external services after startup. In such cases, you don’t want to kill the application, but you don’t want to send it requests either. Kubernetes provides readiness probes to detect and mitigate these situations. A pod with containers reporting that they are not ready does not receive traffic through Kubernetes Services.

>     Note: Readiness probes runs on the container during its whole lifecycle.

> Readiness probes are configured similarly to liveness probes. The only difference is that you use the readinessProbe field instead of the livenessProbe field.
> ```
> readinessProbe:
>   exec:
>     command:
>     - cat
>     - /tmp/healthy
>   initialDelaySeconds: 5
>   periodSeconds: 5
> ```
> Configuration for HTTP and TCP readiness probes also remains identical to liveness probes.

> Readiness and liveness probes can be used in parallel for the same container. Using both can ensure that traffic does not reach a container that is not ready for it, and that containers are restarted when they fail

We will review some examples in this module to understand different options for configuring liveness and readiness probes.

# Configure Liveness Probe
## Configure the Probe

Use the command below to create a directory
```
mkdir -p ~/environment/healthchecks
```
Save the manifest as ~/environment/healthchecks/liveness-app.yaml using your favorite editor. You can review the manifest that is described below. In the configuration file, livenessProbe determines how kubelet should check the Container in order to consider whether it is healthy or not. kubelet uses periodSeconds field to do frequent check on the Container. In this case, kubelet checks liveness probe every 5 seconds. initialDelaySeconds field is to tell the kubelet that it should wait for 5 seconds before doing the first probe. To perform a probe, kubelet sends a HTTP GET request to the server hosting this Pod and if the handler for the servers /health returns a success code, then the Container is considered healthy. If the handler returns a failure code, the kubelet kills the Container and restarts it.
```
cat <<EoF > ~/environment/healthchecks/liveness-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-app
spec:
  containers:
  - name: liveness
    image: brentley/ecsdemo-nodejs
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
EoF
```
Let’s create the pod using the manifest
```
kubectl apply -f ~/environment/healthchecks/liveness-app.yaml
```
The above command creates a pod with liveness probe
```
kubectl get pod liveness-app
```
The output looks like below. Notice the RESTARTS
```
NAME           READY     STATUS    RESTARTS   AGE
liveness-app   1/1       Running   0          11s
```
The kubectl describe command will show an event history which will show any probe failures or restarts.
```
kubectl describe pod liveness-app

Events:
  Type    Reason                 Age   From                                    Message
  ----    ------                 ----  ----                                    -------
  Normal  Scheduled              38s   default-scheduler                       Successfully assigned liveness-app to ip-192-168-18-63.ec2.internal
  Normal  SuccessfulMountVolume  38s   kubelet, ip-192-168-18-63.ec2.internal  MountVolume.SetUp succeeded for volume "default-token-8bmt2"
  Normal  Pulling                37s   kubelet, ip-192-168-18-63.ec2.internal  pulling image "brentley/ecsdemo-nodejs"
  Normal  Pulled                 37s   kubelet, ip-192-168-18-63.ec2.internal  Successfully pulled image "brentley/ecsdemo-nodejs"
  Normal  Created                37s   kubelet, ip-192-168-18-63.ec2.internal  Created container
  Normal  Started                37s   kubelet, ip-192-168-18-63.ec2.internal  Started container
```
## Introduce a Failure

We will run the next command to send a SIGUSR1 signal to the nodejs application. By issuing this command we will send a kill signal to the application process in docker runtime.
```
kubectl exec -it liveness-app -- /bin/kill -s SIGUSR1 1
```
Describe the pod after waiting for 15-20 seconds and you will notice kubelet actions of killing the Container and restarting it.
```
Events:
  Type     Reason                 Age                From                                    Message
  ----     ------                 ----               ----                                    -------
  Normal   Scheduled              1m                 default-scheduler                       Successfully assigned liveness-app to ip-192-168-18-63.ec2.internal
  Normal   SuccessfulMountVolume  1m                 kubelet, ip-192-168-18-63.ec2.internal  MountVolume.SetUp succeeded for volume "default-token-8bmt2"
  Warning  Unhealthy              30s (x3 over 40s)  kubelet, ip-192-168-18-63.ec2.internal  Liveness probe failed: Get http://192.168.13.176:3000/health: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
  Normal   Pulling                0s (x2 over 1m)    kubelet, ip-192-168-18-63.ec2.internal  pulling image "brentley/ecsdemo-nodejs"
  Normal   Pulled                 0s (x2 over 1m)    kubelet, ip-192-168-18-63.ec2.internal  Successfully pulled image "brentley/ecsdemo-nodejs"
  Normal   Created                0s (x2 over 1m)    kubelet, ip-192-168-18-63.ec2.internal  Created container
  Normal   Started                0s (x2 over 1m)    kubelet, ip-192-168-18-63.ec2.internal  Started container
  Normal   Killing                0s                 kubelet, ip-192-168-18-63.ec2.internal  Killing container with id docker://liveness:Container failed liveness probe.. Container will be killed and recreated.
```
When the nodejs application entered a debug mode with SIGUSR1 signal, it did not respond to the health check pings and kubelet killed the container. The container was subject to the default restart policy.
```
kubectl get pod liveness-app
```
The output looks like below
```
NAME           READY     STATUS    RESTARTS   AGE
liveness-app   1/1       Running   1          12m
```
Challenge:

How can we check the status of the container health checks?
Expand here to see the solution
```
kubectl logs liveness-app
```
You can also use kubectl logs to retrieve logs from a previous instantiation of a container with --previous flag, in case the container has crashed
```
kubectl logs liveness-app --previous

<Output omitted>
Example app listening on port 3000!
::ffff:192.168.43.7 - - [20/Nov/2018:22:53:01 +0000] "GET /health HTTP/1.1" 200 16 "-" "kube-probe/1.10"
::ffff:192.168.43.7 - - [20/Nov/2018:22:53:06 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.10"
::ffff:192.168.43.7 - - [20/Nov/2018:22:53:11 +0000] "GET /health HTTP/1.1" 200 17 "-" "kube-probe/1.10"
Starting debugger agent.
Debugger listening on [::]:5858
```

# Configure Readiness Probe
## Configure the Probe

Save the text from following block as ~/environment/healthchecks/readiness-deployment.yaml. The readinessProbe definition explains how a linux command can be configured as healthcheck. We create an empty file /tmp/healthy to configure readiness probe and use the same to understand how kubelet helps to update a deployment with only healthy pods.
```
cat <<EoF > ~/environment/healthchecks/readiness-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: readiness-deployment
  template:
    metadata:
      labels:
        app: readiness-deployment
    spec:
      containers:
      - name: readiness-deployment
        image: alpine
        command: ["sh", "-c", "touch /tmp/healthy && sleep 86400"]
        readinessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 3
EoF
```
We will now create a deployment to test readiness probe
```
kubectl apply -f ~/environment/healthchecks/readiness-deployment.yaml
```
The above command creates a deployment with 3 replicas and readiness probe as described in the beginning
```
kubectl get pods -l app=readiness-deployment
```
The output looks similar to below

```
NAME                                    READY     STATUS    RESTARTS   AGE
readiness-deployment-7869b5d679-922mx   1/1       Running   0          31s
readiness-deployment-7869b5d679-vd55d   1/1       Running   0          31s
readiness-deployment-7869b5d679-vxb6g   1/1       Running   0          31s
```
Let us also confirm that all the replicas are available to serve traffic when a service is pointed to this deployment.
```
kubectl describe deployment readiness-deployment | grep Replicas:
```
The output looks like below
```
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
```
## Introduce a Failure

Pick one of the pods from above 3 and issue a command as below to delete the /tmp/healthy file which makes the readiness probe fail.
```
kubectl exec -it <YOUR-READINESS-POD-NAME> -- rm /tmp/healthy
```
readiness-deployment-7869b5d679-922mx was picked in our example cluster. The /tmp/healthy file was deleted. This file must be present for the readiness check to pass. Below is the status after issuing the command.
```
kubectl get pods -l app=readiness-deployment
```
The output looks similar to below:
```
NAME                                    READY     STATUS    RESTARTS   AGE
readiness-deployment-7869b5d679-922mx   0/1       Running   0          4m
readiness-deployment-7869b5d679-vd55d   1/1       Running   0          4m
readiness-deployment-7869b5d679-vxb6g   1/1       Running   0          4m
```
Traffic will not be routed to the first pod in the above deployment. The ready column confirms that the readiness probe for this pod did not pass and hence was marked as not ready.

We will now check for the replicas that are available to serve traffic when a service is pointed to this deployment.
```
kubectl describe deployment readiness-deployment | grep Replicas:
```
The output looks like below
```
Replicas:               3 desired | 3 updated | 3 total | 2 available | 1 unavailable
```
When the readiness probe for a pod fails, the endpoints controller removes the pod from list of endpoints of all services that match the pod.
Challenge:

How would you restore the pod to Ready status?
Expand here to see the solution

Run the below command with the name of the pod to recreate the /tmp/healthy file. Once the pod passes the probe, it gets marked as ready and will begin to receive traffic again.
```
kubectl exec -it <YOUR-READINESS-POD-NAME> -- touch /tmp/healthy
```
```
kubectl get pods -l app=readiness-deployment
```

# Cleanup

Our Liveness Probe example used HTTP request and Readiness Probe executed a command to check health of a pod. Same can be accomplished using a TCP request as described in the documentation.
```
kubectl delete -f ~/environment/healthchecks/liveness-app.yaml
kubectl delete -f ~/environment/healthchecks/readiness-deployment.yaml
```