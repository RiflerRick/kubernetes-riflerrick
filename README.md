## Commands

```bash
minikube start # for starting the minikube cluster. This starts a VM in the background
# This is used for local setup basically for development. Starts a one node kubernetes cluster

minikube status # shows the cluster status

kubectl run hello-minikube --image=gcr.io/google_containers/echoserver:1.4 --port=8080 # run a container of a specified image in the cluster

kubectl expose deployment hello-minikube --type=NodePort # simply exposing the container

minikube service hello-minikube --url # to get the url where the container can be reached from

```
Minikube stars a local kubernetes cluster solely for development purposes. It stars a single node cluster. Minikube however is not the only tool that enables us to start a kubernetes cluster locally. We can also use docker-client for the purpose of running kubernetes locally. If we do that we would have added a new `context` for kubectl to identify. 

```bash
kubectl config get-contexts # gets all the contexts
```
we can switch to another context using the following command
```bash
kubectl config use-context <name of the context>
```
```bash
kubectl get nodes # for getting all nodes 
```

### Kops
Kubernetes operations

To setup kubernetes as a production cluster on aws, we can use kops. 

Note: for kops to work `aws` cli must be setup in the local machine first and the user configured must have permissions to create ec instances on the aws as kops will attempt to create an aws cluster itself

```bash
kops create cluster --name=kubernetes.riflerrick.tk --state=s3://kops-state-b215b --zones=ap-south-1a --node-count=2 --node-size=t2.micro --master-size=t2.micro --dns-zone=kubernetes.riflerrick.tk 
# The previous command is to run a kubernetes cluster on aws

# name: The name of the cluster in our case is the same as the domain name
# state: the state of the cluster would be stored in an s3 bucket, kops-state-b215b
# zones: the zone where the cluster would be launched is going to be ap-south-1a. This refers to an availability zone in aws
# node-count: number of nodes to be launched. This is excluding the master
# node-size: aws specification of the ec2 instance for the nodes
# master-size: aws specification of the ec2 instance for the master
# dns-zone: dns configured for reaching this cluster

# Note that a custom security group is also created along with cluster.
```
Just after executing the previous command, kops will not immediately create the cluster. kops will allow us to first review the changes that will be made before actually creating the cluster.

```bash
kops edit cluster kubernetes.riflerrick.tk --state=s3://kops-state-b215b

# this will allow us to edit the cluster metadata. The state needs to be provided
```
For creating the cluster we can use the following command

```bash
kops update cluster kubernetes.riflerrick.tk --state=s3://kops-state-b215b --yes

# the state needs to be specified as well
```
The `--yes` option actually applies the changes onto aws. 

Cluster creation may actually take some time. We can validate the cluster using the following command

```bash
kops validate cluster --state=s3://kops-state-b215b
```

Using `kubectl` now we can fetch the nodes that are running in aws.
Make sure that the context of kubectl is pointing to the kops context. The context of kubernetes can be checked using the following command

```bash
kubectl config get-contexts
```
To change the current context use

```bash
kubectl config use-context kubernetes.riflerrick.tk
```

Running a deployment in the cluster
```bash
kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.4 --port=8080
```

The previous command would create a deployment for the image `k8s.gcr.io/echoserver:1.4` and expose the port 8080 in the container where the image would be running.

We can then have a service of type `NodePort` that can expose the running container to the outside world.
```bash
kubectl expose deployment hello-minikube --type=NodePort
```

Next we can get the services using the following command
```bash
kubectl get services
```

This would list out the `NodePort` service. Note that the nodeport service exposes the port only on the worker nodes and not on the master.  

We can delete the cluster using the following command
```bash
kops delete cluster kubernetes.riflerrick.tk --state=s3://kops-state-b215b
```
The previous command prepares the delete operation but does not actually delete the cluster straightaway. For that we need to use the `--yes` option

### Running an app in minikube
---
An example of a kubernetes pod deployment file

```yml
apiVersion: v1
kind: Pod 
metadata:
  name: nodehelloworld.example.com
  labels:
    app: helloworld
spec:
  containers:
  - name: k8s-demo
    image: wardviaene/k8s-demo
    ports:
    - name: nodejs-port
      containerPort: 3000
```

Note here that we are creating a pod by the name of `nodehelloworld.example.com`, The name of the app would be `helloworld` having only one container running the image `wardviaene/k8s-demo`. The exposed port is 3000. 

In order to create this deployment we can use the following command
```bash
kubectl create -f first-app/helloworld.yml # -f is obviously the filename
```

In order to check the pod we can `describe` it.
```bash
kubectl describe pod nodehelloworld.example.com
```

If we actually describe a pod, at the very bottom we will be able to see the events of the pod. The events essentially include events like pulling the image and essentially setting up the container.

From the deployment file we can see that we are going to expose port 3000 of the container. Now we can access that app in 2 ways:
- port-forwarding: 
    ```bash
    kubectl port-forward nodehelloworld.example.com 8081:3000 # basically port 3000 on the pod will be forwarded to port 8081 in the localmachine
    ```

- using a service: 
    ```bash
    kubectl expose pod nodehelloworld.example.com --type=NodePort --name nodehelloworld-service
    ```
    Note that we will need to check the port on which the service exposed the port. We can do so using the following command

    On minikube we can get the ip and the port in the following way:
    ```bash
    minikube service nodehelloworld-service --url
    ```

Once the pod is available we can use the following commands on them:
```bash
kubectl attach nodehelloworld.example.com
# this command is used to attach a terminal to the process running inside the pod. So for instance if we have a django development server running inside the pod, we will be able to see the logs if we attach the pod using kubectl

kubectl exec nodehelloworld.example.com -- ls /app
# with exec we can execute commands inside a running pod

kubectl run -i --tty busybox --image=busybox --restart=Never -- sh
# this would enable us to create another deployment start the pod and run commands in that pod
```   

### Running an app using kops in AWS

helloworld.yml file
```yml
apiVersion: v1
kind: Pod
metadata:
  name: nodehelloworld.example.com
  labels:
    app: helloworld
spec:
  containers:
  - name: k8s-demo
    image: wardviaene/k8s-demo
    ports:
    - name: nodejs-port
      containerPort: 3000
```
It is the same helloworld.yml file as before
One thing to note here is that under the `ports` tag there is a name given. This name can be used later on inside the service. 

Next lets take a look at the service of type `LoadBalancer` that we will use infront of our kubernetes cluster on aws. Such a load balancer when deployed with kops will actually create an ELB on aws.

```yml
apiVersion: v1
kind: Service
metadata:
  name: helloworld-service
spec:
  ports:
  - port: 80 # this is the port of the service
    targetPort: nodejs-port # this is the port on the pod where kubeproxy must connect in order to 
    # find the running containers
    protocol: TCP
  selector:
    app: helloworld
  type: LoadBalancer
```
Note here that we are using the `targetPort` as `nodejs-port`.

There are a few things to note in this file.
- **`ports`**
  - `port`: there are 3 directives under the ports section. The first one defines the port of the service. This means when one service needs to communicate with another service in the same kubernetes cluster, it would use this port.
  - `targetPort`: targetPort is the port on the **pod** where `kubeproxy` must contact in order to find the running containers. In other words if we want our container to be accessible by this service we would have to listen on the targetPort(given that the container has exposed this same port)
  - `protocol`: just protocol

## Concepts

![](https://raw.githubusercontent.com/RiflerRick/kubernetes/master/kube-basics.jpg)

Kubernetes is called a platform of platforms. What this means is that kubernetes provides us with a platform for managing different platforms. It provides an abstraction over the platforms that we generally use for deploying our application to easily manage those platforms. 

Shown in the picture are 2 nodes(machines) with a couple of pods inside them. The atomic unit of kubernetes is a pod. A pod can have multiple containers running inside it. These containers can communicate easily with each other. They can just use the port number to communicate with each other. Pods within a cluster can also communicate with each other but that needs to go over the network. 

On the nodes we also have a kubelet and a kubeproxy service running. kubelet basically enables kubernetes to launch the pods so it connects to the master node in the cluster to get this information. The kubeproxy is going to feed the information about what pods are there in the current node to iptables. iptables is a firewall of linux.

Services are means to communicate between pods. So for instance in the diagram we have a load balancer which is essentially a service. This load balancer service if deployed will essentially spin an elastic load balancer with kops for instance. The load balancer may forward the traffic to any of the nodes' iptables, however the actual pod to connect to may be residing in a different node, it is then the responsibility of iptables to route the traffic to that other node.

So for instance if we were to describe how a pod definition would be in terms of a yaml file:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodehelloworld.example.com
  labels:
    app: helloworld # other kubernetes resources will use this label in order to find this pod
  spec:
    containers:
    - name: k8s-demo
      image: wardviaene/k8s-demo
      ports:
        - containerPort: 3000
```
Under the `spec` the `containers` specification lists the specification of each container that will be run inside the pod. 

### Scaling our pods

if our application is stateless it is possible to horizontally scale it. stateless would mean that our application would not write any local files and it would not keep any session data. If our application would write any data locally that would mean that each pod would be out of sync as each pod is writing data locally and therefore it would not be horizontally scalable. If a request to one pod would yield the same result as making a request to another pod them we can say that our pods are stateless. This obviously means that all databases are stateful as they obviosuly write local files. However in case of web applications, they can be made stateless. Session management and the like can be done outside the container and the web application would be stateless. Any files that need to be saved cannot be saved **locally within the container.

Scaling in kubernetes is done using the Replication Controller. The replication controller will ensure that a specified number of pods will run at all times. Pods created by the replica controller will automatically get replaced if they fail, get deleted or are terminated. For instance going back to our example app, we can use replication controller in the following way:
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: helloworld-controller
spec:
  replicas: 2
  selector:
    app: helloworld # this tells kubernetes that this replication controller is meant for the app 
    # by the label name app as helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
      - name: k8s-demo
        image: wardviaene/k8s-demo
        ports:
        - name: nodejs-port
          containerPort: 3000
``` 
What the replication controller is going to do in this case is that it is going to run the pod defined in it 2 times.
If we now deploy a replication controller using `kubectl create -f helloworld-controller.yaml` given that the name of the replication controller yaml file is `helloworld-controller.yaml`

After deployment we can see the pods using 
```
kubectl get pods
```
This would display all the pods that are running and in this case we would see two pods running as the replication factor given is 2. Now in order to see the status of the pods we can describe the pods using the following command
```
kubectl describe pod <name of the pod>
```
The interesting thing now is that if we try to delete one pod, the replication controller will immediately swing into action and start running another pod to replace the old pod so that we always have 2 healthy pods running at all times.
```
kubectl delete pod <name of the pod>
kubectl get pods
```
on getting the pods we will see that although the status of one pod will be terminating, another pod would have taken its place
It is also possible to scale the pods using the `kubectl scale` command in the following way
```
kubectl scale --replicas=4 -f helloworld-controller.yaml
```
given that the name of the deployment yaml file is `helloworld-controller.yaml`. This would create 4 replicas of the pod.
One thing to note here is that if we had previously running pods and we use the scale command, kubernetes is smart enough not to kill the previously running pods and start them again, but to actually start 2 new pods if 2 pods were already running.
If we wanted to get the name of the replication controller, it can be done in the following way
```
kubectl get rc
```
It is possible to scale our pods using the name of the replication controller as well in the following way
```bash
kubectl get rc
kubectl scale --replicas=4 rc/<name of the replication controller> # the number of replicas here is absolute, which means that if 4 replicas were already running, it would not do anything, 
kubectl scale --replicas=1 rc/<name of the replication controller> # this would terminate 3 replicas and just run one
```
It is however worth mentioning one more time that replication is always possible in stateless applications.
```
kubectl delete rc/<name of the replication controller>
```
### Replica Set

Replica Set is the next generation replication controller, it supports a new selector that can do selection based on filtering according to a set of values. Replica Set is actually used by the deployment object.

### Deployment

A deployment declaration in k8s allows you to do app deployments and updates. When using the deployment object, you define the **`state`** of the application. Kubernetes will then make sure the cluster matches the desired state. This is fundamental to the whole idea of kubernetes and exactly the reason why kubernetes is what it is.

With a deployment object the following things can be done
- Create a deployment
- Update a deployment
- Do rolling updates(zero downtime deployments)
- Roll back to a previous version of the app
- pause/resume a deployment(that would mean that we want to roll out only a certain percentage of the running pods)

An example of a deployment can be
```yaml
apiVersion: extension/v1beta1
kind: Deployment
metadata:
  name: helloworld-deployment
spec:
  replicas: 3 # 3 pods will be run
  template: 
    metadata:
      labels:
        app: helloworld
    spec: # pod specifications
      containers:
      - name: k8s-demo
        image: wardviaene/k8s-demo
        ports:
        - containerPort: 3000 # recall that this is the exact port on the container that has been exposed
```
Useful commands on deployments
- get current deployments
  ```
  kubectl get deployments
  ```
- get information of replica sets
  ```
  kubectl get rs
  ```
- show pods with labels
  ```
  kubectl get pods --show-labels
  ```
- getting the deployment status
  ```bash
  kubectl rollout status deployment/hellworld-deployment # given that the name of the deployment is helloworld-deployment
  ```
- we can change our image in a deployment in the following way
  ```bash
  kubectl set image deployment/<name of the deployment> k8s-demo=k8s-demo:2 # this would change the image k8s-demo to k8s-demo version 2
  ```
- we can edit a deployment in the following way
  ```
  kubectl edit deployment/<name of the deployment>s
  ```
- getting the history of our deployed versions can be done in the following way
  ```
  kubectl rollout history deployment/<name of the deployment>
  ```
- rollback to a previous version
  ```
  kubectl rollout undo deployment/<name of the deployment>
  ```
- rolling back to a specified version using the following command
  ```
  kubectl rollout undo deployment/<name of the deployment> --to-version=n
  ```
   other than using `deployment/<name of the deployment>` it is also possible to do 
   `deployment <name of the deployment>`

Now if we wanted to expose our deployment we would do that using a `NodePort` service in the following way
```bash
kubectl expose deployment <name of the deployment> --type=NodePort
kubectl get service # would show that a nodeport type service was created
kubectl describe service <name of the service>
```
On describing the service this is what we would get back
```
Name:                     helloworld-deployment
Namespace:                default
Labels:                   app=helloworld
Annotations:              <none>
Selector:                 app=helloworld
Type:                     NodePort
IP:                       10.108.200.233
Port:                     <unset>  3000/TCP
TargetPort:               3000/TCP
NodePort:                 <unset>  32762/TCP
Endpoints:                172.17.0.4:3000,172.17.0.5:3000,172.17.0.6:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```
In order to see the url from which we can access the pod, we can use the following command
```
minikube service <name of the service> --urls
```
Note: during creation of a deployment we can use the option `--record` in order to see the changes as well. 

By default the kubernetes deployment revision history is set to 2 which means only 2 entries will be shown in the revision history. We can change that by editing the deployment. We need to add the following key under deployment `spec`
`revisionHistoryLimit`.

### Service

Basically for accessing pods

An example of a service definition might be like the following
```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld-service # name of the service
spec:
  ports:
  - port: 31001
    nodePort: 31001 # port on the node from where i would be able to access the pod
    targetPort: nodejs-port # targetPort is the port on the pod where the app is running
    protocol: TCP
  selector: 
    app: helloworld
  type: NodePort
```
In a similar fashion used for creating deployments, we can create services
```bash
kubectl create -f <name of the service yaml file>
```
Specifying a nodeport is not mandatory, if not specified, k8s will choose a random port that is not conflicting with any other application. If however we specify the nodeport this conflict management has to be done by ourselves.

```
kubectl get svc # svc is short for service
```
If we go ahead and describe the service, here is what we get
```
Name:                     helloworld-deployment
Namespace:                default
Labels:                   app=helloworld
Annotations:              <none>
Selector:                 app=helloworld
Type:                     NodePort
IP:                       10.108.200.233
Port:                     <unset>  3000/TCP
TargetPort:               3000/TCP
NodePort:                 <unset>  32762/TCP
Endpoints:                172.17.0.4:3000,172.17.0.5:3000,172.17.0.6:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```
here **IP** is the ip of the service, **Port** is the port of the service where we can access the pods
The service obviosuly is inside the cluster which means we cannot directly access the service from the external world. 
The **TargetPort** is the port on the pod where the container is running and the container is listening to
The **NodePort** is the port from which we can access pod and hence the container and the app

In order to get more clarity we can do the following

```bash
rajdeep@adminstrator-Latitude-3590:~/Desktop/kubernetes  (master) $ minikube ssh
$ docker ps | grep k8s-demo
b049a5aab2d5        wardviaene/k8s-demo          "/bin/sh -c 'npm sta…"   2 hours ago         Up 2 hours                              k8s_k8s-demo_helloworld-deployment-67bd889c49-nsz4k_default_ddbfaf69-9d79-11e8-a805-08002738fe97_0
0dc9cadbc992        wardviaene/k8s-demo          "/bin/sh -c 'npm sta…"   2 hours ago         Up 2 hours                              k8s_k8s-demo_helloworld-deployment-67bd889c49-gw96n_default_ddbf8fc1-9d79-11e8-a805-08002738fe97_0
6528ccc29874        wardviaene/k8s-demo          "/bin/sh -c 'npm sta…"   2 hours ago         Up 2 hours                              k8s_k8s-demo_helloworld-deployment-67bd889c49-r2j99_default_dd9af936-9d79-11e8-a805-08002738fe97_0
$ curl http://10.108.200.233:3000
Hello World!$ 
$ # kubectl describe service helloworld-deployment
$ # the above command would return the following endpoints 172.17.0.4:3000,172.17.0.5:3000,172.17.0.6:3000
$ # these are infact the ips of the pods
$ curl http://172.17.0.4:3000
Hello World!$ 
$ # we can also get inside the container and access it 
$ docker exec -it b049a5aab2d5 bash
root@helloworld-deployment-67bd889c49-nsz4k:/app# curl http://localhost:3000   
Hello World!root@helloworld-deployment-67bd889c49-nsz4k:/app#
root@helloworld-deployment-67bd889c49-nsz4k:/app# 
root@helloworld-deployment-67bd889c49-nsz4k:/app# 
root@helloworld-deployment-67bd889c49-nsz4k:/app# exit
exit
$ # since we are inside the k8s cluster, we can directly access the pod using the nodeport
$ curl http://localhost:32762    
Hello World!$ 
$
```
The ip of the service also changes if we create a new service after deleting the old one. It is however also possible to keep that fixed as well

### Labels

just like tags in aws.
With labels we can have pods deployed on specific nodes and not just any node.
The first thing to do in such a case would be to add a label or multiple labels to our nodes. 
```bash
kubectl label nodes node1 hardware=high-spec
kubectl label nodes node2 hardware=low-spec
```
once the nodes are labelled we can have the pods get deployed to specific nodes, in the following way
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodehelloworld.example.com
  labels:
    app: helloworld
spec:
  containers:
  - name: k8s-demo
    image: wardviaene/k8s-demo
    ports:
    - containerPort: 3000
  nodeSelector: # this is the directive that selects the node with this key value pair
    hardware: high-spec
```

### Healthchecks

2 types of ways to run healthchecks
- running a command in the container periodically
- periodic checks on a url(http)

Healthchecks in kubernetes can be implemented using the **liveness probe** in the following way:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodehelloworld.example.com
  labels:
    app: helloworld
spec:
  containers:
  - name: k8s-demo
    image: wardviaene/k8s-demo
    ports:
    - containerPort: 3000
    livenessProbe:
      httpGet:
        path: /
        port: 3000
      initialDelaySeconds: 15 # initial delay
      timeoutSeconds: 30
```
Similarly we have the readiness probe that checks if the pod is ready to take requeuts. If the readiness probe fails then the IP address will be removed from the service until the readiness probe on the pod succeeds. For the livenesss probe, if the liveness probe fails then the container will be restarted.

### Pod state

pods have a status field which find when we do a `kubectl get pods`. When a pod is in the **running** state, it means:
- the pod has been bound to a node
- all containers have been created
- at least one container is still runnning or is starting/restartng

Other valid statuses are:
- **pending**: the pod has been accepted but is not running. May happen when the container image is still downloading. if the pod cannot be scheduled because of resource constraints it will also be in this status. 

- **succeeded**: All containers in the pod have been terminated and will not be restarted.

- **failed**: All containers in the pod have been terminated and atleast one container returned a failure code. The failure code is simply the exit code of the process when the container terminates.

- **unknown**: This means that the status could not be determined, this might happen when the node on which the pod is supposed to run may be down.

There are 5 different types of pod conditions:

- **PodScheduled**: pod has been scheduled to a node
- **Ready**: pod is ready to serve requests and is going to added to matching services
- **Initialized**: initialization containers have been started successfully
- **Unschedulable**: The pod can't be scheduled may be due to resource constraints
- **ContainersReady**: All containers in the pod are ready

The **container state** can be got from the following command:
```bash
kubectl get pod <pod_name> -o yaml
```
The container state can be running, terminated or waiting

### Pod Lifecyle

![](https://raw.githubusercontent.com/RiflerRick/kubernetes/master/k8s_pod_lifecycle.png)
The following diagram shows us the
