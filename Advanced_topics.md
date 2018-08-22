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

Its an alternative to the load balancer(that is also available as a service). Ingress controllers allows us to easily expose services that need to be accessible from outside the cluster.

There are default ingress controllers available and it is also possible to create our own ingress controller.

The way this works any traffic from the internet hits the ingress controller first at whichever port is defined for incoming traffic. The ingress controller is a service that is associated with a pod(that runs a container for instance an nginx container). The ingress controller then passes on the traffic to the respective services. 

An example of an ingress controller can be the following:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress # ingress object
metadata:
    name: helloworld-rules
spec:
    rules:
    - host: helloworld-v1.example.com
      http:
        paths:
        - path: /
          backend:
            serviceName: helloworld-v1 # so basically for the host helloworld-v1.example.com route 
            # traffic to the path / at the service helloworld-v1 at port 80
            servicePort: 80
    - host: helloworld-v2.example.com
      http:
        paths:
        - path: /
          backend:
            serviceName: helloworld-v2 # similarly for the other service
            servicePort: 80
```

Note that along with the Ingress object it is also necessary to have an ingress controller(which runs as a pod with a container).

The ingress controller only works for HTTP/s protocols.

## External DNS

external dns automatically creates the necessary DNS records in your DNS server(for instance route53). For every hostname that we create in our ingress object it automatically creates a DNS entry in our DNS server.

Note that the external DNS runs as a separate pod altogether.

## Running Stateful apps

### Volumes

Volumes in kubernetes allows us to store data outside the container(similar to docker volumes for instance).

Volumes can be attached using different volume plugins. It is possible to store data in local volumes that exist on the same node. It is also possible to store data on something like EBS(elastic block storage) instances on aws.

If our node stops working, the pod can be rescheduled on another node, the volume(EBS storage for instance) can be then attached to that node automatically. However this only works if the EBS instance is in the same AZ as the nodes themselves.

Creating volumes(aws volumes on EBS) using aws cli:

```bash
aws ec2 create-volume --size 10 region us-east-1 --availability-zone us-east-1a --volume-type gp2
```
10 gb size az us-east-1a.

Once this volume is created it will have a volume id that we need to use for creating the pod's volumes:

```yaml
spec:
    containers:
    .
    .
    .
    volumes:
    - name: myvol # /myvol now becomes a path inside
    awsElasticBlockStore:
        volumeID: <vol id of aws EBS>
```

### Volumes Autoprovisioning

kubernetes plugins have the capability of provisioning storage for you. This means that the AWS plugin for instance can provision storage for us by creating the volumes in AWS before attaching them to a node.

So we do not need to create a volume manually using awscli anymore. This is done using the StorageClass object.

For instance:

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
    name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
    type: gp2 # general purpose 2
    zone: us-east-1
```

Next we need to create a persistent volume claim and specify the size

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
    name: myclaim
    annotations:
        volume.beta.kubernetes.io/storage-class: "standard" # refers to the standard storage class 
        # that was created earlier
spec:
    accessModes:
        - ReadWriteOnce # i can have only one EBS volume that I can read and write to at the same time
        # ReadWriteOnce simply means I can read and write at the same time.
    resources:
        requests:
            storage: 8Gi
```

Now that we have both the storage class and the persistent volume claim declared we can use the volume in our pods in the following way:

```yaml
kind: Pod
.
.
.
spec:
    containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
    volumes:
    - name: mypd
    persistentVolumeClaim:
        claimName: myclaim # this was the exact claim name that was given in the PersistentVolumeClaim 
        # object
```

It is also possible to have efs(elastic file system) volumes but they cannot be auto provisioned
