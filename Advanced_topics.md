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

It is also possible to have efs(elastic file system) volumes but they cannot be auto provisioned.

### Pod Presets

pod presets can inject Kubernetes resources like secrets, configmaps, volumes and environment variables at runtime. 

So lets say we have 20 applications we want to deploy and they all need to get a specific credential. One way to do this would be to edit the deployment of all these 20 applications to have the configmaps and secrets in them or we could have a pod preset which will inject the configmaps and secrets to all matching pods.

When injecting environment variables and volume mounts, the pod preset will apply the changes to all containers within the pod.

Example of a pod preset:

```yaml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
    name: share-credential
spec:
    selector:
        matchLabels:
            app: myapp
    env:
        - name: MY_SECRET
          value: "12345"
    volumeMounts:
        - mountPath: /share
          name: share-volume
    volumes:
        - name: share-volume
          emptyDir: {}
```
It is totally possible to use more than one pod presets and they will all be applied to matching pods. If there is a conflict, the PodPreset will not be applied to the pod

### StatefulSets

StatefulSets were introduced to run stateful applications that need a stable pod hostname(podname). Pods are in general given random names in the fashion `podname-<random string>`. If statefulsets are used pod names will have a sticky identity and the pod names will not change even after the pod is rescheduled. Statefulsets allow stateful apps stable storage with volumes based on their ordinal number(podname-x).

The obvious question that can arise is the application of such a feature. Statefulsets are used in applications where there is a cluster and the application itself needs to know the actual hostname of each and every pod in the cluster. For instance, if we are setting up cassandra as a cluster, cassandra requires the hostname of the nodes(in our case pods) of the cluster as a configuration setting. Now if the hostnames are changing it does not make sense to set up cassandra because it will never work.

### Daemon Sets

DaemonSets ensures that every single node in the k8s cluster runs the same pod resource. This is actually useful if we want to ensure that a certain pod is running on every single kubernetes node. When a node is added to the cluster, a new pod will be started automatically. Similarly when the node is removed, the pod will **not** be rescheduled on another node.

Typical use cases are:

- log aggregation(for having all logs in a central place we would need to have a pod running on each and every node).
- monitoring
- load balancers/reverse proxies/API gateways
- any other use case where we need to have a pod running on every node

### Resource Usage Monitoring

Heapster enables container cluster monitoring and performance analysis. It provides a monitoring platform for kubernetes. It is a pre-requisite if you want to do pod autoscaling in kubernetes. Heapster exports cluster metrics via REST endpoints. There are many different backends that we can use with Heapster. A backend is simply the place where the monitoring data is stored. For instance InfluxDB.

For resource usage monitoring, the yaml files can be found in the github repository of Heapster. 

This is how a typical structure of resource monitoring may look like:

![](https://raw.githubusercontent.com/RiflerRick/kubernetes/master/k8s-resource-monitoring.png)

There is a kubernetes process called cAdvisor that gathers information about the metrics directly from the pod and sends all that information to the heapster pod, heapster is then going to save this to influxDB and after that graphana will be used in order to show all the information

Heapster is however depricated recently and therefore it is better to use something like prometheus to do that

### Horizontal Pod Autoscaling

In kubernetes possible to configure auto scaling of deployments, replication controllers or replica sets.

It is possible to scale based on CPU usage out of the box, however if we want to have custom metrics like average request latency or the queries per second or the like, we need start the cluster with the environment variable `ENABLE_CUSTOM_METRICS` to be true.

Autoscaling will periodically query the utilization for the targeted pods. By default 30 seconds is the time in which the periodic query will happen however it can be changed using the `--horizontal-pod-autoscaler-sync-period` when launching the controller manager in the master node. 

Autoscaling will use heapster the monitoring tool to gather its metrics.

For this to work in our deployment we will have a resource request in the following way:

```yaml
spec:
    containers:
    .
    .
    .
        resources:
            requests:
                cpu: 200m
```

Example of an HPA:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
    name: hpa-example
spec:
    scaleTargetRef:
        apiVersion: extensions/v1beta1
        kind: Deployment # the deployment needs to scale
        name: hpa-example
    minReplicas: 1
    maxReplicas: 10
    targetCPUUtilizationPercentage: 50 # 50 percent of the CPU that we requested, so basically since
    # we requested 200m, the targetCPUUtilization would be 100m
```

### Affinity/Anti-affinity

Just like we have node selector for scheduling pods on specific nodes, we can have more expressive rules for scheduling pods using affinity/anti-affinity.

It is possible to create rules that are not hard requirements but rather a preferred rule meaning that the scheduler will still be able to schedule the pod even if the rules are not met.

It is possible to have pod affinity as well(other than node affinity). For instance we can have a particular affinity defined that 2 pods will never be on the same node. Affinity and anti-affinity is relevant only during scheduling so for instance if we have a pod that is already running and we spin up a node that turns out to be a better match for the pod, the pod will not get automatically scheduled to the other node, we would need to manually delete the pod and create it again.

- Node affinity:

    - **requiredDuringSchedulingIgnoredDuringExecution**: hard requirement. rules need to be met before the pod could be scheduled

    - **preferredDuringSchedulingIgnoredDuringExecution**: a little more loose requirement. This will try to enfore the rule but not actually gurantee it.

Affinity can be defined in the deployment spec itself:

```yaml
spec:
    affinity:
        nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                    - key: env
                      operator: In
                      values:
                      - dev
            preferredDuringSchedulingIgnoredDuringExecution:
                - weight: 1 # the higher the weight the more preference is given to that rule
                  preference:
                    matchExpressions:
                    - key: team
                      operator: In
                      values: 
                      - engineering-project1
    containers:
    .
    .
    .
```

**Concept of weight**
Lets say we have n number of rules and for a particular node, 2 of those rules match with weights 2 and 5. In that case the total score is 2+5=7, lets say for another node 3 of the rules match however the weights of the rules are 2,1 and 3 which leads to a total score of 2+1+3=6. Since the previous node got a higher score, the pod will be scheduled on that node

In addition to the labels that can be added, there are also pre-populated labels that can be added for node matching, for instance:

- kubernetes.io/hostname
- failure-domain.beta.kubernetes.io/zone
- failure-domain.beta.kubernetes.io/region
- beta.kubernetes.io/instance-type
- beta.kubernetes.io/os
- beta.kubernetes.io/arch

We can label a node using the following command:

```bash
kubectl label node <node name> env=dev # the label key value pair being env, dev
```

- Pod affinity and Anti affinity

a good use case of pod affinity and anti affinity would be the following:

- there may be a requirement where one pod needs to be co-located with another pod(i.e in the same node). In such a case scenario we are going to use pod affinity.
- For instance if we have an app that uses redis as cache then we might want redis to be always in the same node as the pod itself.
- It is also possible to have pods co-located within the same AZ.

When writing the pod affinity and anti-affinity rules we need to specify a topology domain, called a `topologyKey` in the rules.

The topology key refers to a node label.

If the affinity rule matches then the new pod will only be scheduled on the node that has the same topology Key as the node in which the current pod is running.

![](https://raw.githubusercontent.com/RiflerRick/kubernetes/master/k8s-pod-affinity.png)

Lets consider the above scenario. The new pod has an affinity set to the appname which is myapp. This initial match is for matching the pod against which the co-location match is going to be done. Since the selector given here is `app` having values `myapp`. The pod in node 2 is chosen for comparison. However for selecting the node the topologyKey would be used which in this case is the kubernetes hostname that the pod(original) has.

![](https://raw.githubusercontent.com/RiflerRick/kubernetes/master/k8s-pod-affinity-2.png)

Similarly a second situation might be for the AZ.

**Pod Anti-Affinity**
Anti-affinity can be used to make sure that a pod is only scheduled once in a node.

Basically anti-affinity dictates that if a pod label matches, k8s will not schedule on that node. This is basically useful when we are trying to separate pods and avoid running them on the same node.

![](https://raw.githubusercontent.com/RiflerRick/kubernetes/master/k8s-pod-anti-affinity.png)

When writing pod affinity rules, its possible to use the following operators:

- In/NotIn (does a label have one of these values)
- Exists/DoesNotExist (does a label exist or not)

```
Note: Interpod affinity currently requires a substantial level of processing so if there are too many nodes this feature might be used with caution.
```

### Taints and Tolerations

Toleration is the opposite of node affinity. Tolerations allow a node to repel a set of pods. Taints mark a node, tolerations are applied to pods to influence the scheduling of pods. 

For instance the master has a taint: `node-role.kubernetes.io/master:NoSchedule`

For adding a new taint on a node you can use:

```bash
kubectl taint nodes node1 key=value:NoSchedule
```

The above taint will make sure that no pods are scheduled on node1 unless they have a matching toleration.

For instance if we have a deployment that has a toleration of the form

```yaml
tolerations:
- key: "key"
  operator: "Equal" # we can use Equal(using key and value) or Exists(only using key)
  value: "value"
  effect: "NoSchedule"
```

**NoSchedule**: a hard requirement that a pod will not be scheduled unless there is a matching toleration.
**PreferNoSchedule**: k8s will try and avoid placing a pod that does not have a matching toleratio but it is not a hard requirement
**NoExecute**: if this taint is applied, the pods running on that node will be stopped immediately if a matching toleration is not found

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoExecute"
  tolerationSeconds: 3600 # the pod will run for 3600 seconds before being evicted. If this value
  # is not given the toleration will match and the pod will keep on running on the tainted node
```

