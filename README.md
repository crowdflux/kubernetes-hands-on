# Kuberenets Workshop

- Goto https://www.katacoda.com/courses/kubernetes/playground
- Click on "start scenario" button.
- You will be presented with an UI providing terminal access to a new kubernetes cluster.
- Zoom in. And use the master node terminal for further steps.

## Pre-requisites
- Basic knowledge about Docker and Kubernetes.
- Go through the Playment's kubernetes documentation @ https://www.notion.so/playment/Playment-on-Kubernetes-ee29a0d5aca04f4ebde6e04f6dc2e97d
- Download the 'kubernetes-hands-on' git repository from playment org.
	`git clone https://github.com/crowdflux/kubernetes-hands-on.git`

### Get to know about your kubernetes cluster

- Know about your kubernetes api server
	`kubectl cluster-info`
	The above command will give you details about your kubernetes api server, whether it is running or not and the api endpoint on which it would be listening to.
- List the nodes in your cluster
	`kubectl get nodes`
- List all the pods inside your cluster
	`kubectl get pods --all-namespaces `
- List the namespaces in your cluster
	`kubectl get namespaces`
- See what pods are running in *kube-system* namespace
	`kubectl get pods --namespace kube-system `
	you can also use short forms `po` for `pods` and `-n` in place of `--namespace` like the below command
	`kubectl get po -n kube-system`
- Try deleting some pods and see what happens
	Run the below command on master.
	`kubectl get pods --all-namespaces -o wide -w`
	Open another terminal on master node and try deleting some pods and see what happens.
	`kubectl delete pod [pod-name] -n [namespace]`

## Lets deploy our first application to kubernetes

### Let us create a pod for the application
- Lets deploy the hello application with it's docker image  `paulbouwer/hello-kubernetes:1.5`
- Go to the folder `kubernetes-hands-on/my-first-deployment` in the downloaded git repository.
- Run the below command to create a new pod for the hello application.
	`kubectl create -f pod.yaml`
- See if your pod got created.
	`kubectl get pods`
	Run describe command for more info on your pod
	`kubectl describe pod [POD-NAME]`
	Let's see if the hello application is responding to your api requests
	`kubectl get pod -o wide`
	On the node server terminal, run `curl [POD-IP]:[PORT]`

- Let us have one more pod for the same application.
	 Run the create command one more time.
	 `kubectl create -f pod.yaml`
	 Oops - a pod is already present with the name `hello-kubernetes-custom-1`. 
	 You gotta create a new yaml file with the same config and update the name to `hello-kubernetes-custom-2` and try creating the second pod
	 Run `kubectl create -f pod-2.yaml`
	 now check whether there are two pods running. `kubectl get pods -o wide`
	 
	 You might require more number of pods to handle high load on your application. But this requires you to write more number of YAML files which is not desirable. **ReplicaSets** comes to your rescue.
 

### Let us create a replicaset

1. Clean up the old pods.
`kubectl delete pod hello-kubernetes-custom-1 hello-kubernetes-custom-2`
1. Let us create a single pod with replicaset.
	`kubectl create -f replicaset.yaml`
1. Check the replicaset status and check if the pod is running.
	`kubectl get rs`

	`kubectl get pods`
1. Let us now scale the number of pods to 3.
	`kubectl scale rs hello-kubernetes-custom --replicas=3`
	
	check if there are 3 pods running. `kubectl get pods`

	Clean up the resources and let's move on to next thing.
	`kubectl delete rs hello-kubernetes-custom`

### Create a Deployment for your app

1. `kubectl create -f deployment.yaml`
1. Let's update the application deployment
	`kubectl edit deployment hello-kubernetes-custom`
	Update the value of the environment variable **MESSAGE** to "I just deployed this on Kubernetes!"
	Save the changes and exit.
1. Check the status of the update by running `kubectl rollout status deployment hello-kubernetes-custom`
	You should see a new pod getting created and the old pod getting terminated.
	Verify the deployment by making an api call to the application pod.
1. Let's try rolling back the deployment changes.
	` kubectl rollout history deployment hello-kubernetes-custom`
	Now let's rollback to the old version
	`kubectl rollout undo deployment hello-kubernetes-custom --to-revision=1`

### Expose the application to internet

1. Expose the application inside kubernetes cluster as a service
	1. `kubectl apply -f service-clusterIP.yaml`
	1. `kubectl get service`
	1. `kubectl describe svc hello-kubernetes-custom-1`
	1. curl [service-ip]:[service-port]
1. Expose the application to internet using a public LoadBalancer
	1. `kubectl apply -f service-loadbalancer.yaml`
	1. `kubectl get svc`
	1. `kubectl describe svc hello-kubernetes-custom-2`
	1. curl [loadbalancer]:[loadbalancer-port]
1. Debug the service
	`kubectl get pods,svc -o wide --show-labels`
	`kubectl get endpoints hello-kubernetes-custom-1`

### Let us deploy an nginx server using helm

** Let us take a look at nginx helm chart @ https://github.com/bitnami/charts/tree/master/bitnami/nginx**

1. Initialize helm client
	`kubectl apply -f tiller-rbac.yaml`
	`helm init --service-account tiller`
2. Add the helm repository
	`helm repo add bitnami https://charts.bitnami.com/bitnami`
	`helm repo list`
3. Install the nginx helm chart
	`helm install bitnami/nginx`
4. Let's update the helm release.
	`helm ls`
	
	`helm upgrade [RELEASE-NAME] bitnami/nginx --set replicaCount=2`
5. Let's rollback the helm release
	`helm history [RELEASE-NAME]`
	`helm rollback [RELEASE-NAME] 0`


### communication between applications inside kubernetes
`kubectl get svc -n nginx`
`kubectl exec -it [pod-name]`
`curl http://[nginx-svc-name].[namespace]`

### Autoscaling

Install metrics server
`helm install stable/metrics-server --name metrics-server --version 2.0.4 --namespace kube-system`

Check the status of the metrics server
`kubectl get apiservice v1beta1.metrics.k8s.io -o yaml`

Deploy a php sample application
`kubectl run php-apache --image=k8s.gcr.io/hpa-example --requests=cpu=50m --expose --port=80`

Create an autoscaler
`kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10`

`kubectl get hpa`

Let's generate the load
`kubectl run -i --tty load-generator --image=busybox /bin/sh`

`while true; do wget -q -O - http://php-apache; done`

Open another terminal and wathc the hpa scale the pods
`kubectl get hpa -w`
