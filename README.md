# Kubernetes Deployment Examples

This repo contains simple kubernetes examples for teaching purposes.

## Notes

## The command line - practicalities

- Install oh my zsh or equivalent.
- Docker completion - <https://docs.docker.com/compose/completion/>
- Do the same for kubernetes and make aliases! - <https://thorsten-hans.com/autocompletion-for-kubectl-and-aliases-using-oh-my-zsh>
  - K = kubectl
  - Docker-compose = dc
  - etc.

## Terminology

- Kube control or kubectl
- K8s = K12345678s = kubernetes
- Node = single server in the kubernetes cluster
- Kubelet = kubernetes agent running on nodes
- Control plane= a set of containers that manage the cluster
  - Includes api server, scheduler, controller manager, etcd, coreDNS, and more.
- Pods
  - pod is one more containers running together on one node
  - Containers are alway in pods
  - Can have volumes etc.
- Controller
  - For creating/updating pods and other objects.
  - Deployment and ReplicaSet (and statefulset, daemonset, job, cronjob etc.)
- Service
  - A network endpoint to connect to a pod
  - A persistant connection point in the cluster
- Namespace
  - A filter of your view at the command line (filtered group of objects in the cluster)

## Basic Commands

See more [here](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

- Kubectl run (pod creation - so k run mynginx —image nginx
- Kubectl create (create some resources via cli or yaml)
- Kubectl apply (create/update anything via yaml)
- kubectl get nginx
- Kubectl get all (give all info!! 😃 )
- k describe pod/my-apache-5d589d69c7-ltdd6
- Scaling
  - kk scale deployment --replicas 3 my-apache

## Abstractions

Deployment -> replicates -> pod.N -> vol Nic -> container (nginx)

## Services

- Commands
- K expose - a consistent endpoint so something else can change behind.

### Types of Services

- ClusterIP (default) 
  - only available in cluster. (Nodes and pods)
  - Gets its own DNS address. 
  - Single virtual ip allocated
  - Pods can reach service on apps port number
- NodePort
  - Designed for something outside of the cluster to communicated inside.
  - Creates a high port on each node
- LoadBalancer
  - Controls a load balancer endpoint external to the cluster
  - Creates cluster IP and nodeport
  - Tells the hosting provider to provide IP.
- ExternalName
  - Not often used
  - Stuff in your cluster needs to talk to something external, so your cluster can resolve name external services. Using coredns

### Examples

#### Basic Example

```bash
# for watching changes to the configuration
watch -n1 kubectl get all
# create a deployment
k create deployment httpenv --image=bretfisher/httpenv
# change the scale from 1 to 5
k scale deployment httpenv --replicas 5
# expose the deployment/port to the rest of the cluster
k expose deployment httpenv --port 8888
# see the createsd service like this
k get service
# lets run a container inside the environment to see if we can connect (at this point you cannot connect from your own host)
k run --generator run-pod/v1 tmp-shell --rm -it --image bretfisher/netshoot -- bash
curl <the ip from service get from before>:8888
# now lets try a nodeport 
k expose deployment httpenv --port 8888 --name httpenv-np --type NodePort
#you can curl it now on the machine.
#lets try and actually make a load balancer in front of all the services
k expose deployment httpenv --port 8888 --name httpenv-lb --type LoadBalancer
# see the newly added load balancer
k get service
#web go to port and refresh (gets same host)
#now lets try from out local command line
curl localhost:31377 #(notice you get different host each time)
# Now do the same for 8888
watch -n1 curl -s localhost:8888
```

#### Elastic search example

```bash
# create a deployment of elastic search
k create deployment search --image=elasticsearch:2
# scale it to 5 instances
k scale deployment search --replicas 5
# create a load balancer
k expose deployment search --port 9200 --name search --type LoadBalancer
# watch how we connect to different hosts with
watch -n1 curl localhost:9200
# Go to to : http://localhost:9200 and refresh - why is this the result?
k scale deployment search --replicas 1
# What happened to the curl? Yes, now we only get the same result.
```

#### Elastic search example using yml

See [search.yml](./yml-examples/search.yml). To run it, do the following.

```bash
# move to the directory with the yml examples
cd yml-examples
# make kubernetes apple the config in search.yml
k apply -f search.yml
# try and update the replica set and run the above command again.
# To delete it run:
k delete -f search.yml
# try deleting a pod and see it recreate it:
k delete pod app-elasticsearch-deployment-<your-pod-name>
```

In this example, get "kind" types from running `k api-resources`, and look in the column KIND. Get versions by `k api-versions`. Here you can for example see the apps/v1 version.

To see what the changes of an action would be, run `k apply -f search.yml --dry-run`. The output would be:

```bash
service/app-elasticsearch-service created (dry run)
deployment.apps/app-elasticsearch-deployment created (dry run)
```

This shows that the changes would be to create a service, and a deployment resource.

It is also possible to output the yaml with `k apply -f search.yml --dry-run -o yaml`. This can be used with most commands, such as `k create deployment search --image=elasticsearch:2 --dry-run -o yaml`, which would output:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: search
  name: search
spec:
  replicas: 1
  selector:
    matchLabels:
      app: search
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: search
    spec:
      containers:
      - image: elasticsearch:2
        name: elasticsearch
        resources: {}
status: {}
```

The YAML specs can also be explored with: `k explain services --recursive`. This can be changed to see all options in other things than services, such as deployment with `k explain deployment --recursive`. Mind you, this is big, but can be a help for finding variables that are otherwise hidden away. To make it more managable, you can go further into the specs by dotting through the sections such as `k explain services.spec.type` which gives a better explanation as seen below:

```text
KIND:     Service
VERSION:  v1

FIELD:    type <string>

DESCRIPTION:
     type determines how the Service is exposed. Defaults to ClusterIP. Valid
     options are ExternalName, ClusterIP, NodePort, and LoadBalancer.
     "ExternalName" maps to the specified externalName. "ClusterIP" allocates a
     cluster-internal IP address for load-balancing to endpoints. Endpoints are
     determined by the selector or if that is not specified, by manual
     construction of an Endpoints object. If clusterIP is "None", no virtual IP
     is allocated and the endpoints are published as a set of endpoints rather
     than a stable IP. "NodePort" builds on ClusterIP and allocates a port on
     every node which routes to the clusterIP. "LoadBalancer" builds on NodePort
     and creates an external load-balancer (if supported in the current cloud)
     which routes to the clusterIP. More info:
     https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
```

## DNS

Move between namespaces with: \<hostname>.\<namespace>.svc.cluster.local

## Other tools

- [Kompose](https://kompose.io) is a good converter for old docker-compose files.
- [Helm](https://helm.sh) - a package manager.
- [Stern](https://github.com/wercker/stern) for better log tailing
- [CI Tools of choice](https://github.com/wercker/stern) - merge yaml files to master and deploy automatically.

## Scaling networking patterns

For types of strategies look [here](https://kemptechnologies.com/load-balancer/load-balancing-algorithms-techniques/). and 

### Poor mans load balancer

Basically a DNS alias round robin.

```bash
docker example with —network-alias=search
docker network create mynet
docker run -d --name e1 --network=mynet --network-alias=search elasticsearch:2
docker run -d --name e2 —network=mynet --network-alias=search elasticsearch:2
docker run -it --network=mynet alpine nslookup search
docker run -it --network=mynet centos curl -s search:9200
```

### K8s load balancer

See the [example in the earlier section](####Elastic-search-example-using-yml).

### Registry Server
