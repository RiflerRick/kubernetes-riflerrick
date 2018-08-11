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

if our application is stateless it is possible to horizontally scale it. stateless would mean that our application would not write any local files and it would not keep any session data. If our application would write any data locally that would mean that each pod would be out of sync as each pod is writing data locally and therefore it would not be horizontally scalable. If a request to one pod would yield the same result as making a request to another pod them we can say that our pods are stateless. This obviously means that all databases are stateful as they obviosuly write local files. However in case of web applications, they can be made stateless. Session management and the like can be done outside the container and the web application would be stateless. Any files that need to be saved cannot be saved locally within the container.

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
