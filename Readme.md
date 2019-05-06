# Introduction
Here are the notes of my understanding of Kubernetes and their sources.

# How To

Here a basic walk-through to make your app works:

## Deploy an App on Google Cloud

1. Database : Create or Select one
2. Cluster : Create or Select one
3. Docker Registry Secret : Register your Docker Private Registry if needed
4. ConfigMaps : Create your Configuration (Laravel .env)
5. YAML (Deploy, Service...)
6. Ingress

## Deploy an App in MiniKube
1. Create a deployment (= docker run) Or Resource Controller (= docker service)
2. Create a service = publish ports and maps to your deployment(s)/rc(s) inside the cluster
3. Create an ingress = L7 HTTP(s) LoadBalancer which map ip/url/port to your services or use NodePort

## Use a Persitent Volume in a Pod/Service

1. Create a Persistent Volume or Select one
2. Create a Persistent Volume Claim or Select one
3. Defined a named volume to the selected volume claim and Map your named volume to your path in the container

# Kubernetes 101

Here are some basics commands.

## GET Cluster IP of a service
```
kubectl get services/webapp1-clusterip-svc -o go-template='{{(index .spec.clusterIP)}}'
```

where webapp1-clusterip-svc is the name of your service

## Get Pod instance name
```
kubectl get pods --selector="name=frontend" --output=jsonpath={.items..metadata.name}
```

where frontend is the name of your service

## Cluster System Pods List
```
kubectl get pods -n kube-system
```
-n stands for namespace

## Cluster status
```
kubectl cluster-info
```

## Cluster nodes list
```
kubectl get nodes
```

## List 
### persistent volumes
```
kubectl get pv
```

### persitent volume claims
```
kubectl get pvc
```

### env variables of a pod
```
kubectl exec -it POD_NAME env
```

where POD_NAME is the name of your pod

## Connect to clusters

### My current connection is ?
```
kubectl config current-context
```

### List available connections
```
kubectl config get-clusters
```

### Switch to minikube
```
kubectl config use-context minikube
```

## Do like docker

If you come from Docker Swarm, here some equivalent Docker commands.

### docker ps
```
kubectl get pods
```

### docker service
```
kubectl get deployments
```

### docker inspect
```
kubectl describe pod NAME
```

### docker network ls

```
kubectl get services|svc
```

** Service is confusing in kubernetes... (read its definition) **

# Theory

## Resource Controller

The replication controller defines how many instances should be running, the Docker Image to use, and a name to identify the service. Additional options can be utilized for configuration and discovery.

If Redis were to go down, the replication controller would restart it on an active node.

## Deployment

Deployments are a newer and higher level concept than Replication Controllers. They manage the deployment of Replica Sets (also a newer concept, but pretty much equivalent to Replication Controllers), and allow for easy updating of a Replica Set as well as the ability to roll back to a previous deployment.

Previously this would have to be done with kubectl rolling-update which was not declarative and did not provide the rollback features.

## Service

Kubernetes Services are an abstract that defines a policy and approach on how to access a set of Pods. The set of Pods accessed via a Service is based on a Label Selector.

A Kubernetes service is a named load balancer that proxies traffic to one or more containers. The proxy works even if the containers are on different nodes.

Services proxy communicate within the cluster and rarely expose ports to an outside interface.

When you launch a service it looks like you cannot connect using curl or netcat unless you start it as part of Kubernetes. The recommended approach is to have a LoadBalancer service to handle external communications.

### Type (Of Service)

#### Cluster IP

Accessible ONLY from INSIDE the cluster = PRIVATE IP

Cluster IP is the default approach when creating a Kubernetes Service. The service is allocated an internal IP that other components can use to access the pods.

By having a single IP address it enables the service to be load balanced across multiple Pods.

```
apiVersion: v1
kind: Service
metadata:
  name: my-internal-service
spec:
  selector:
    app: my-app
  type: ClusterIP
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
```

#### Target Port

Target ports allows us to separate the port the service is available on from the port the application is listening on. TargetPort is the Port which the application is configured to listen on. Port is how the application will be accessed from the outside.

##### externalIPs

Accessible from OUTSIDE the cluster (external world) ~= PUBLIC IP

Another approach to making a service available outside of the cluster is via External IP addresses.

### NodePort

Accessible from OUTSIDE the cluster (external world) for ports 30000-32767 ~= PUBLIC IP

NodePort allows you to set well-known ports that are shared across your entire cluster. This is like -p 80:80 in Docker.

When would you use this?
There are many downsides to this method:

  1. You can only have once service per port
  2. You can only use ports 30000–32767
  3. If your Node/VM IP address change, you need to deal with that

```
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  selector:
    app: my-app
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    nodePort: 30036
    protocol: TCP
```

### Load Balancer

When running in the cloud, such as EC2 or Azure, it's possible to configure and assign a Public IP address issued via the cloud provider. This will be issued via a Load Balancer such as ELB. This allows additional public IP addresses to be allocated to a Kubernetes cluster without interacting directly with the cloud provider.

## Pods

One pod can contain 1 or more containers :
 - pause container : manager network for the main container
 - main container : your image

### States
With all controllers and services defined Kubernetes will start launching them as Pods. A pod can have different states depending on what's happening. For example, if the Docker Image is still being downloaded then the Pod will have a pending state as it's not able to launch. Once ready the status will change to running.

List of all pods like "docker ps"
```
kubectl get pods
```

More info on a pod like "docker inspect CONTAINER"
```
kubectl describe pod NAME
```

## Ingress
An Ingress enables inbound connections to the cluster, allowing external traffic to reach the correct Pod.

Ingress is actually NOT a type of service. Instead, it sits in front of multiple services and act as a “smart router” or entrypoint into your cluster.
You can do a lot of different things with an Ingress, and there are many types of Ingress controllers that have different capabilities.

The default GKE ingress controller will spin up a HTTP(S) Load Balancer for you. This will let you do both path based and subdomain based routing to backend services. For example, you can send everything on foo.yourdomain.com to the foo service, and everything under the yourdomain.com/bar/ path to the bar service.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  backend:
    serviceName: other
    servicePort: 8080
  rules:
  - host: foo.mydomain.com
    http:
      paths:
      - backend:
          serviceName: foo
          servicePort: 8080
  - host: mydomain.com
    http:
      paths:
      - path: /bar/*
        backend:
          serviceName: bar
          servicePort: 8080
```

## Persistent Volume

For Kubernetes to understand the available NFS shares, it requires a PersistentVolume configuration. The PersistentVolume supports different protocols for storing data, such as AWS EBS volumes, GCE storage, OpenStack Cinder, Glusterfs and NFS. The configuration provides an abstraction between storage and API allowing for a consistent experience.

In the case of NFS, one PersistentVolume relates to one NFS directory. When a container has finished with the volume, the data can either be Retained for future use or the volume can be Recycled meaning all the data is deleted. The policy is defined by the persistentVolumeReclaimPolicy option.

For structure is:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <friendly-name>
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: <server-name>
    path: <shared-path>
```

The spec defines additional metadata about the persistent volume, including how much space is available and if it has read/write access.

### Claim a volume
Once a Persistent Volume is available, applications can claim the volume for their use. The claim is designed to stop applications accidentally writing to the same volume and causing conflicts and data corruption.

The claim specifies the requirements for a volume. This includes read/write access and storage space required. An example is as follows:

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: claim-mysql
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
#### Warning : Pod claiming a volume
If a Persistent Volume Claim is not assigned to a Persistent Volume, then the Pod will be in Pending mode until it becomes available. In the next step, we'll read/write data to the volume.


## Secrets
Kubernetes requires secrets to be encoded as Base64 strings.

Using the command line tool we can create the Base64 strings and store them as variables to use in a file. 

```
username=$(echo -n "admin" | base64)
password=$(echo -n "a62fjbd37942dcs" | base64)
```

The secret is defined using yaml. Below we'd using the variables defined above and providing them with friendly labels which our application can use. This will create a collection of key/value secrets that can be accessed via the name, in this case test-secret

```
echo "apiVersion: v1
kind: Secret
metadata:
  name: test-secret
type: Opaque
data:
  username: $username
  password: $password" >> secret.yaml
```

The result

```
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
type: Opaque
data:
  username: YWRtaW4=
  password: YTYyZmpiZDM3OTQyZGNz
```

This yaml file can be used to with Kubectl to create our secret. When launching pods that require access to the secret we'll refer to the collection via the friendly-name.

### Create the secret
Use kubectl to create our secret.

```
kubectl create -f secret.yaml
```

The following command allows you to view all the secret collections defined.

```
kubectl get secrets
```

### Consuming secrets
To populate the environment variable we define the name, in this case SECRET_USERNAME, along with the name of the secrets collection and the key which containers the data.

The structure looks like this:

```
- name: SECRET_USERNAME
valueFrom:
 secretKeyRef:
   name: test-secret
   key: username
```

## ConfigMaps

"docker config" Like

Analyse config contents :
```
kubectl get configmap myapp-env -o yaml
```

SOURCES : 
 - https://kubernetes.io/docs/tutorials/configuration/
 - https://medium.com/@xcoulon/managing-pod-configuration-using-configmaps-and-secrets-in-kubernetes-93a2de9449be

### Consume via Volumes
The use of environment variables for storing secrets in memory can result in them accidentally leaking. The recommend approach is to use mount them as a Volume.

To mount the secrets as volumes we first define a volume with a well-known name, in this case, secret-volume, and provide it with our stored secret.

```
volumes:
 - name: secret-volume
   secret:
     secretName: test-secret
```

When we define the container we mount our created volume to a particular directory. Applications will read the secrets as files from this path.

```
volumeMounts:
 - name: secret-volume
   mountPath: /etc/secret-volume
```

Once started you can interact with the mounted secrets. For example, you can list all the secrets available as if they're regular data. 

For example 
```
kubectl exec -it secret-vol-pod ls /etc/secret-volume
```

Reading the files allows us to access the decoded secret value. To access username we'd use 
```
kubectl exec -it secret-vol-pod cat /etc/secret-volume/username
```

# How to use Docker Private Registry with Kubernetes

When you pull images from Docker Private Registry with native Docker, you can do the authentication with docker login. However, if you're using these images from Kubernetes, you can't run docker login command directly. You'll need to create a Secret Key instead as follows:

```
$ kubectl create secret docker-registry myregistrykey --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

You'll get a response like this:
```
secret "myregistrykey" created
```

Once it's done, you can add in your pod definition the imagePullSecrets section :
```
spec:
  template:
    ...
      imagePullSecrets:
      - name: myteam-gitlab
```

Here is a complete sample :
```
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: awesomeapps
spec:
  containers:
    - name: foo
      image: my/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
```

Note:

I had a problem that kubectl describe pod ... says "Back-off pulling image "DOCKERREGISTRYSERVER/image:latest". I could fix it just by deleting the secret and creating it again. You can delete "myregistrykey" as follows:
```
$ kubectl delete secret myregistrykey
```
# Tools

## kubeless
With Kubeless you can deploy functions without the need to build containers. These functions can be called via regular HTTP(S) calls or triggered by events submitted to message brokers like Kafka.

Kubeless aims to be an open source FaaS solution to clone the functionalities of AWS Lamdba/Goole Cloud Functions.

## HELM

[HELM](heml.sh) is a package manager like apt to install :
 - apps (like redis, mysql...)
 - kubernetes plugins like (ingress-nginx)[]

SOURCES :

 - https://coderwall.com/p/c7j_5q/how-to-use-docker-private-registry-with-kubernetes
 - https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/

# MiniKube

[Minikube](https://github.com/kubernetes/minikube) is perfect to play locally with a Cluster.
I use it to test my deployments, services before send it to Google Cloud.

Here are some basic commands :

## Open the dashboard
```
minikube dashboard
```

## Get IP Addr
```
minikube ip
```

## Enable Ingress
```
minikube addons enable ingress
```

[Setting up ingress on minikube](https://medium.com/@Oskarr3/setting-up-ingress-on-minikube-6ae825e98f82)

# Kompose

Do you need to convert docker-compose yaml file to Kubernetes configuration files ? Use the right tool : [Kompose](http://kompose.io)

Use it like docker-compose :
```
kompose up # start from a docker-compose.yaml
kompose down # halt from a docker-compose.yaml
```

Or to convert a docker-compose.yaml file :  
```
kompose convert
```

# References

 *  [KataCoda](https://www.katacoda.com) a really great plateform to learn by using real env in your browser !
 *  [GCP Kubernetes Tutorials](https://cloud.google.com/kubernetes-engine/docs/tutorials/?authuser=1)
 *  [Nodeport vs Loadbalancer vs Ingress when should I use what](https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0)
 *  [Redis Replication](https://redis.io/topics/replication)
