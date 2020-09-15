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
watch -n1 kubectl get all
k create deployment httpenv --image=bretfisher/httpenv
k scale deployment httpenv --replicas 5
k expose deployment httpenv --port 8888
K get service
k run --generator run-pod/v1 tmp-shell --rm -it --image bretfisher/netshoot -- bash
curl ip:8888
k expose deployment httpenv --port 8888 --name httpenv-np --type NodePort
k expose deployment httpenv --port 8888 --name httpenv-lb --type LoadBalancer
k get service (to get port)
#web go to port and refresh (gets same host)
curl localhost:31377 #(gets different host each time)
# Now do the same for 8888 (or watch -n1 curl localhost:8888)
```

#### Elastic search example

```bash
k create deployment search --image=elasticsearch:2
k scale deployment search --replicas 5
k expose deployment search --port 9200 --name search --type LoadBalancer
watch -n1 curl localhost:9200
# Go to to : http://localhost:9200 and refresh - why is this the result?
k scale deployment search --replicas 1
# What happened to the curl?
```

#### Elastic search example using yml

See [search.yml](./yml-examples/search.yml). To run it, do the following.

```bash
cd yml-examples
k apply -f search.yml
# try and update the replica set and run the above command again.
# To delete it run:
k delete -f search.yml
# try deleting a pod and see it recreate it:
k delete pod app-elasticsearch-deployment-<your-pod-name>
```

## DNS

Move between namespaces with: \<hostname>.\<namespace>.svc.cluster.local

## Other tools

- [Kompose](https://kompose.io) is a good converter for old docker-compose files.
- [Helm](https://helm.sh) - a package manager.
- [Stern](https://github.com/wercker/stern) for better log tailing
- [CI Tools of choice](https://github.com/wercker/stern) - merge yaml files to master and deploy automatically.

### Scaling networking patterns

#### Poor mans load balancer

Basically a DNS alias round robin.

```bash
docker example with —network-alias=search
docker network create mynet
docker run -d --name e1 --network=mynet --network-alias=search elasticsearch:2
docker run -d --name e2 —network=mynet --network-alias=search elasticsearch:2
docker run -it --network=mynet alpine nslookup search
docker run -it --network=mynet centos curl -s search:9200
```

#### K8s load balancer

See [search.yml](./yml-examples/search.yml)

#### Registry Server
