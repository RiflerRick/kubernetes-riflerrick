### Commands

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

#### Kops
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

