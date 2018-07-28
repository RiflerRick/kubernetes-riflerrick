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