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
