# Deploy the Official Kubernetes Dashboard
The official Kubernetes dashboard is not deployed by default, but there are instructions in the official documentation

We can deploy the dashboard with the following command:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```
Since this is deployed to our private cluster, we need to access it via a proxy. Kube-proxy is available to proxy our requests to the dashboard service. In your workspace, run the following command:
```
kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &
```
This will start the proxy, listen on port 8080, listen on all interfaces, and will disable the filtering of non-localhost requests.

This command will continue to run in the background of the current terminal’s session.



# Access the Dashboard

Now we can access the Kubernetes Dashboard

    In your Cloud9 environment, click Tools / Preview / Preview Running Application
    Scroll to the end of the URL and append:

/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

Open a New Terminal Tab and enter
```
aws eks --region us-east-1 get-token --cluster-name eksworkshop-eksctl | jq -r '.status.token' 
```
Copy the output of this command and then click the radio button next to Token then in the text field below paste the output from the last comman