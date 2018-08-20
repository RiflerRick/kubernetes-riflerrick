## Service Discovery using DNS

As of kubernetes 1.3 DNS is a built in service that comes with kubernetes. The addons are in /etc/kubernetes/addons directory on the master node. The DNS service can be used within pods to find other services that are running on the **same cluster**.

Multiple containers in a pod can contact each other directly and so do not need any service.

A container in the same pod can connect the port of the other container directly using localhost:port. So if there are 2 containers running in the same pod, they will be able to contact each other without any external help. So if for instance we have a web server and a database service, running in the same pod then obviously for the web server to contact the database service we do not need any other service(kubernetes service).

When a container lets say in a pod wants to connect to another container in another pod, this is the time we need service discovery and we need to have service definitions for each pod, basically services integrated with each pod so that they can talk to each other.

![](https://raw.githubusercontent.com/RiflerRick/kubernetes/master/k8s_service_discovery.png)

For instance lets say we have the above 2 pods running 2 applications. If an application wants to talk to another application. On both the pods we have a container running. The first pod has an associated service having an IP 10.0.0.1 and the second pod has a service having an IP 10.0.0.2. 

If we execute the following command on the first pod:
```bash
host app1-service # we would get back the ip address of the service
# here note that app1 is the name of the service and we can see that we essentially need simply the 
# name of the service to get the ip address of the service
host app2-service
# we would get back the ip address of the second service. This would obviously only work if both the pods are in the same namespace
host app2-service.default
# here we are explicitly specifying the default namespace
host app2-service.default.svc.cluster.local
# this is what is called the fully qualified domain name
```

### How it works

![](https://raw.githubusercontent.com/RiflerRick/kubernetes/master/k8s-service-discovery-internals.png)

So there is a kube-dns pod that also runs whenever kubernetes is running. This pod runs in a separate namespace: `kube-system` and is exposed as a service called the kube-dns service that has an ip.

## ConfigMaps

Configuration parameters that are not secrets can be put as configmaps.
For creating configmaps
```bash
kubectl create configmap app-config --from-file=./app/app.properties # for instance
```
it is possible to use configmaps as volumes
it is also possible to use configmaps as environment variables

## Ingress Controller