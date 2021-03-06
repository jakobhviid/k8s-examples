# Kubernetes Deployment Examples

This repo contains simple kubernetes examples for teaching purposes.

## Notes

## The command line - practicalities

To make it easier to navigate the CLI of docker, docker-compose and kubernetes, first make sure autocompletion works in your installation.

- Install oh my zsh or equivalent.
- Docker completion - <https://docs.docker.com/compose/completion/>
- Do the same for kubernetes and make aliases! - <https://thorsten-hans.com/autocompletion-for-kubectl-and-aliases-using-oh-my-zsh>
  - k = kubectl
  - d = docker
  - docker-compose = dc
  - etc.

Most commands in theis section uses `k`instead of `kubectl`. Be aware of this when running the commands.

## Terminology

- Kube control or kubectl
- K8s = K12345678s = kubernetes
- Node = single server in the kubernetes cluster
- Kubelet = kubernetes agent running on nodes
- Control plane = a set of containers that manage the cluster
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
  - A persistent connection point in the cluster
- Namespace
  - A filter of your view at the command line (filtered group of objects in the cluster)

## Basic Commands

See more [here](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

- kubectl run (pod creation - so k run mynginx —image nginx
- kubectl create (create some resources via cli or yaml)
- kubectl apply (create/update anything via yaml)
- kubectl get nginx
- kubectl get all (give all info!! 😃 )
- kubectl describe pod/my-apache-5d589d69c7-ltdd6
- Scaling
  - kubectl scale deployment --replicas 3 my-apache

## Abstractions

Deployment -> replicates -> pod.N -> vol Nic -> container (nginx)

## Services

Services always exist in the context of a deployment/pod. Therefore `k expose` is used in the context of a service to create a consistent endpoint for it.

### Types of Services (networking)

- ClusterIP (default)
  - only available in cluster. (Nodes and pods)
  - Gets its own DNS address.
  - Single virtual ip allocated
  - Pods can reach service on apps port number
- NodePort
  - Designed for something outside of the cluster to communicated inside.
  - Creates a high port on each node
- LoadBalancer
  - Controls a load balancer endpoint external to the cluster.
  - Creates cluster IP and NodePort automatically in the background.
  - Tells the hosting provider to provide IP.
- ExternalName
  - Not often used.
  - Stuff in your cluster needs to talk to something external, so your cluster can resolve name external services. Using CoreDNS.

More information on how this works can be found in these excellent [slides](https://speakerdeck.com/thockin/kubernetes-a-very-brief-explanation-of-ports).

#### DNS

Move between namespaces with: \<hostname>.\<namespace>.svc.cluster.local

## Examples

### Basic Example

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

# Note: to add more more forwards to an existing load balancer use the following:
k patch svc kubernetes-dashboard -p '{"spec":{"externalIPs":["10.10.10.253"]}}'
```

### Elastic search example

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

### Elastic search example using yml

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

## Creating your YAML files

Side note, YAML can also be applied from url's:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
```

### Digging through the documentation using CLI

In this example, get "kind" types from running `k api-resources`, and look in the column KIND. Get versions by `k api-versions`. Here you can for example see the apps/v1 version.

To see what the changes of an action would be, run `k apply -f search.yml --dry-run`. The output would be:

```bash
service/app-elasticsearch-service created (dry run)
deployment.apps/app-elasticsearch-deployment created (dry run)
```

This shows that the changes would be to create a service, and a deployment resource.
Be aware that they added `--dry-run=client` and `--dry-run=server`, which is the new way of making dry runs. Without `=server` it would run the client version of the dry run, which does not take the current status of the deployed resources into account.

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

## Labels and Annotations

They are a list of `key: values` that can be used for selecting and grouping pods. It could be `teir: frontend`, `env: prod`, or `customer: acme corp`. For example, a load balancer could be set up to only hit the `prod` environment. An example could be `k get pods -l env=prod`, which would match any pods with a label of `prod`. You can also match multiple labels with `k get pods -l env=prod,tier=frontend`.
It is possible to use the `-l` filter on an several things like `k apply -f file.yaml -l env=prod`, which would apply only services/pods with specific labels.

Labels are set in the metadata section of a YAML spec.

## Other tools

- [Kompose](https://kompose.io) is a good converter for old docker-compose files.
- [Helm](https://helm.sh) - a package manager.
- [Stern](https://github.com/wercker/stern) for better log tailing
- CI Tools of choice - merge yaml files to master and deploy automatically.

## Scaling networking patterns

For types of strategies look [here](https://kemptechnologies.com/load-balancer/load-balancing-algorithms-techniques/).

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

### Registry Server pattern

TODO: update this section.
