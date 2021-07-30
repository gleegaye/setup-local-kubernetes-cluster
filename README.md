## setup local kubernetes cluster using docker
 
1.   Download and install kind binary by using this [link](https://kind.sigs.k8s.io/docs/user/quick-start/)

`NOTE: the documentation is available at the link given above`


I use the package manager for windows (chocolatey) to install the kind package

```shell
choco install kind
```

2.  Create a cluster with a single node
```shell
λ  kind create cluster                                                 
Creating cluster "kind" ...                                            
 • Ensuring node image (kindest/node:v1.21.1) �  ...                   
 ✓ Ensuring node image (kindest/node:v1.21.1) �                        
 • Preparing nodes �   ...                                             
 ✓ Preparing nodes �                                                   
 • Writing configuration �  ...                                        
 ✓ Writing configuration �                                             
 • Starting control-plane �️  ...                                      
 ✓ Starting control-plane �️                                           
 • Installing CNI �  ...                                               
 ✓ Installing CNI �                                                    
 • Installing StorageClass �  ...                                      
 ✓ Installing StorageClass �                                           
Set kubectl context to "kind-kind"                                     
You can now use your cluster with:                                     
                                                                       
kubectl cluster-info --context kind-kind                               
                                                                       
Have a nice day! �                                                    
```
3. Getting access to you cluster

```shell
λ  kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:52435
CoreDNS is running at https://127.0.0.1:52435/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
3.1. list your cluster node
```shell
λ  kubectl get nodes -o wide
NAME                 STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION                CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane,master   17h   v1.21.1   172.18.0.2    <none>        Ubuntu 21.04   4.19.128-microsoft-standard   containerd://1.5.2
```
from now we have a single node tha uses containerd, let's take a look what's happing inside our control-plane docker container.

** check out the container name/id
```shell
λ  docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED        STATUS        PORTS                       NAMES
7b3d159549de   kindest/node:v1.21.1   "/usr/local/bin/entr…"   18 hours ago   Up 18 hours   127.0.0.1:52435->6443/tcp   kind-control-plane
```
now we get its name , lets get into the container:

```shell
root@kind-control-plane:/# which crictl                                                                                                                
/usr/local/bin/crictl                                                                                                                                  
root@kind-control-plane:/# crictl ps                                                                                                                   
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID                   
ef97694311117       296a6d5035e2d       18 hours ago        Running             coredns                   0                   153525402ec66            
768e3a5c51586       296a6d5035e2d       18 hours ago        Running             coredns                   0                   b14c69ab7fc83            
ab3b2506984fc       e422121c9c5f9       18 hours ago        Running             local-path-provisioner    0                   378288baf5260            
12f05e44a5dae       6de166512aa22       18 hours ago        Running             kindnet-cni               0                   50b1f93d99eff            
6b6777f2e177d       0e124fb3c695b       18 hours ago        Running             kube-proxy                0                   215763f1bba4e            
386b60da6dea1       0369cf4303ffd       18 hours ago        Running             etcd                      0                   6412ce5be5d05            
21b3f2811920d       94ffe308aeff9       18 hours ago        Running             kube-apiserver            0                   c7518e21d511c            
aa6ec7318170e       96a295389d472       18 hours ago        Running             kube-controller-manager   0                   0f1545492da31            
5e36b8bbc7ac8       1248d2d503d37       18 hours ago        Running             kube-scheduler            0                   83fb85ce90b1a            
root@kind-control-plane:/#                                                                                                                                                                                                                                                                               
```
Those are the list of containers running in the kube-container `kind-control-plane`. we have all the kubernets stuff within the kind container. that's kubernets in docker. This is possible because, all those containers are using the container runtime **containerd** directly. **This is Kubernetes In Docker(Kind)**

All the kubernetes config files and the static manifests can be find in **/etc/**
```shell
root@kind-control-plane:/# ls -l /etc/kubernetes                        
total 36                                                                
-rw------- 1 root root 5578 Jul 29 14:26 admin.conf                     
-rw------- 1 root root 5598 Jul 29 14:26 controller-manager.conf        
-rw------- 1 root root 1946 Jul 29 14:27 kubelet.conf                   
drwxr-xr-x 1 root root 4096 Jul 29 14:26 manifests                      
drwxr-xr-x 3 root root 4096 Jul 29 14:26 pki                            
-rw------- 1 root root 5546 Jul 29 14:26 scheduler.conf                 
root@kind-control-plane:/# ls -l /etc/kubernetes/manifests/             
total 16                                                                
-rw------- 1 root root 2189 Jul 29 14:26 etcd.yaml                      
-rw------- 1 root root 3826 Jul 29 14:26 kube-apiserver.yaml            
-rw------- 1 root root 3349 Jul 29 14:26 kube-controller-manager.yaml   
-rw------- 1 root root 1384 Jul 29 14:26 kube-scheduler.yaml            
root@kind-control-plane:/#                                              
```
you can see that we have the **etcd,apiserver,controller-manager and the scheduler** manifests. this is a proper kubernetes running inside docker.

4. Deploying our first pod

if you don't know how to write a pod manifest, don't worry, we are going to tell to kubernetes to generate it four us. Then we can save that manifest in a file named **nginx_pod.yaml**

Before that , we are going to create a namespace where we deploy the pod.

```shell
λ  kubectl create namespace test
namespace/test created
```
```shell
λ  kubectl get ns
NAME                 STATUS   AGE
default              Active   18h
kube-node-lease      Active   18h
kube-public          Active   18h
kube-system          Active   18h
local-path-storage   Active   18h
test                 Active   13s
```
```shell
λ  kubectl run -name nginx --image=nginx --dry-run=client -o yaml          
apiVersion: v1                                                             
kind: Pod                                                                  
metadata:                                                                  
  creationTimestamp: null                                                  
  labels:                                                                  
    run: nginx                                                             
  name: nginx                                                              
  namespace: test                                                           
spec:                                                                      
  containers:                                                              
  - image: nginx                                                           
    name: nginx                                                            
    resources: {}                                                          
  dnsPolicy: ClusterFirst                                                  
  restartPolicy: Always                                                    
status: {}                                                                 
```

This is the pod manifest, lets save it a file tha we could we use for versionning.
```shell
λ  kubectl run -name nginx --image=nginx --dry-run=client -o yaml > C:\Users\u162842\Documents\k8s\nginx-pod.yaml
```
now, we can easily create a pod by the good way.
```shell
λ  kubectl create -f  /Documents/k8s/nginx-pod.yaml
pod/nginx created
```
list our existing pods (all pods in all namespaces)

```shell
λ  kubectl get po -A
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-558bd4d5db-6knqf                     1/1     Running   0          18h
kube-system          coredns-558bd4d5db-ndbh6                     1/1     Running   0          18h
kube-system          etcd-kind-control-plane                      1/1     Running   0          18h
kube-system          kindnet-rgl25                                1/1     Running   0          18h
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          18h
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          18h
kube-system          kube-proxy-9ph5x                             1/1     Running   0          18h
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          18h
local-path-storage   local-path-provisioner-547f784dff-f6qwh      1/1     Running   0          18h
**test                 nginx                                        1/1     Running   0          2m19s**
```
The pod we've just create in the test namespace:
```shell
λ  kubectl get po -n test
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          5m36s
```
Kind can also deploy a High Available (HA) kubernetes infrastructure. if you take a look in the documentation (link at the top), you could see that it's pretty easy to deploy that type of infrasture with kind.

This is from kind website:
```yaml
# a cluster with 3 control-plane nodes and 3 workers
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
```
5. Deploy HA cluster
lets delete the first cluster
```shell
λ  kind delete cluster
Deleting cluster "kind" ...
```
The cluster is completely deletes:
```shell
λ  docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
There is no running container.

Now, we are going to deploy our HA kubernetes cluster
```shell
λ  kind create cluster --config  /Documents/k8s/my-cluster.yaml
Creating cluster "kind" ...
 • Ensuring node image (kindest/node:v1.21.1) �  ...
 ✓ Ensuring node image (kindest/node:v1.21.1) �
 • Preparing nodes � � � � � �   ...
 ✓ Preparing nodes � � � � � �
 • Configuring the external load balancer ⚖️  ...
 ✓ Configuring the external load balancer ⚖️
 • Writing configuration �  ...
 ✓ Writing configuration �
 • Starting control-plane �️  ...
 ✓ Starting control-plane �️
 • Installing CNI �  ...
 ✓ Installing CNI �
 • Installing StorageClass �  ...
 ✓ Installing StorageClass �
 • Joining more control-plane nodes �  ...
 ✓ Joining more control-plane nodes �
 • Joining worker nodes �  ...
 ✓ Joining worker nodes �
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! �
```
lets check 

```shell
λ  kubectl get nodes
NAME                  STATUS   ROLES                  AGE     VERSION
kind-control-plane    Ready    control-plane,master   5m46s   v1.21.1
kind-control-plane2   Ready    control-plane,master   4m23s   v1.21.1
kind-control-plane3   Ready    control-plane,master   3m24s   v1.21.1
kind-worker           Ready    <none>                 3m2s    v1.21.1
kind-worker2          Ready    <none>                 3m2s    v1.21.1
kind-worker3          Ready    <none>                 3m1s    v1.21.1
```
Since we have three master nodes, we are required to have a load-balancer and kind hnow that. Tt deploys and additionnal docker container for us when it was deploying the nodes we specify in our kind-config file.

```shell
λ  docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED         STATUS         PORTS                       NAMES
**c28970711aba   kindest/haproxy:v20200708-548e36db   "/docker-entrypoint.…"   8 minutes ago   Up 8 minutes   127.0.0.1:49868->6443/tcp   kind-external-load-balancer**
295ec0b973cf   kindest/node:v1.21.1                 "/usr/local/bin/entr…"   8 minutes ago   Up 8 minutes                               kind-worker3
b1a3a592cc85   kindest/node:v1.21.1                 "/usr/local/bin/entr…"   8 minutes ago   Up 8 minutes   127.0.0.1:49871->6443/tcp   kind-control-plane3
8e01cd48adc6   kindest/node:v1.21.1                 "/usr/local/bin/entr…"   8 minutes ago   Up 8 minutes   127.0.0.1:49870->6443/tcp   kind-control-plane
473ca5d66752   kindest/node:v1.21.1                 "/usr/local/bin/entr…"   8 minutes ago   Up 8 minutes                               kind-worker
5447f4a745b1   kindest/node:v1.21.1                 "/usr/local/bin/entr…"   8 minutes ago   Up 8 minutes   127.0.0.1:49869->6443/tcp   kind-control-plane2
241120741a44   kindest/node:v1.21.1                 "/usr/local/bin/entr…"   8 minutes ago   Up 8 minutes                               kind-worker2
```
Now lets take a look inside the load-balancer container. 

```shell
λ  docker exec -it  kind-external-load-balancer sh
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 haproxy -sf 7 -W -db -f /usr/local/etc/haproxy/haproxy.cfg
   29 root      0:04 haproxy -sf 7 -W -db -f /usr/local/etc/haproxy/haproxy.cfg
   40 root      0:00 sh
   48 root      0:00 ps
/ #
```
you see that we have a haproxy. We are now going look at the haproxy config file
```shell
/ # cat /usr/local/etc/haproxy/haproxy.cfg                                                                              
# generated by kind                                                                                                     
global                                                                                                                  
  log /dev/log local0                                                                                                   
  log /dev/log local1 notice                                                                                            
  daemon                                                                                                                
                                                                                                                        
resolvers docker                                                                                                        
  nameserver dns 127.0.0.11:53                                                                                          
                                                                                                                        
defaults                                                                                                                
  log global                                                                                                            
  mode tcp                                                                                                              
  option dontlognull                                                                                                    
  # TODO: tune these                                                                                                    
  timeout connect 5000                                                                                                  
  timeout client 50000                                                                                                  
  timeout server 50000                                                                                                  
  # allow to boot despite dns don't resolve backends                                                                    
  default-server init-addr none                                                                                         
                                                                                                                        
frontend control-plane                                                                                                  
  bind *:6443                                                                                                           
                                                                                                                        
  default_backend kube-apiservers                                                                                       
                                                                                                                        
backend kube-apiservers                                                                                                 
  option httpchk GET /healthz                                                                                           
  # TODO: we should be verifying (!)                                                                                    
                                                                                                                        
  server kind-control-plane kind-control-plane:6443 check check-ssl verify none resolvers docker resolve-prefer ipv4    
  server kind-control-plane2 kind-control-plane2:6443 check check-ssl verify none resolvers docker resolve-prefer ipv4  
  server kind-control-plane3 kind-control-plane3:6443 check check-ssl verify none resolvers docker resolve-prefer ipv4  
/ #                                                                                                                     
```
we can see in that particular file that any connecction comming into the **frontend (frontend control-plane )** will be directly distributed to one of the 3 control-plane to the port **6443**

` server kind-control-plane kind-control-plane:6443 check check-ssl verify none resolvers docker resolve-prefer ipv4    
  server kind-control-plane2 kind-control-plane2:6443 check check-ssl verify none resolvers docker resolve-prefer ipv4  
  server kind-control-plane3 kind-control-plane3:6443 check check-ssl verify none resolvers docker resolve-prefer ipv4  `

You may notice that kind uses the names of the containers instead of their ip addresses. This is possible because of the internal docker network.

```shell
λ  docker network ls
NETWORK ID     NAME       DRIVER    SCOPE
405ef54d2a9e   bridge     bridge    local
53d91e7ab8d8   host       host      local
**e97e87ca16da   kind       bridge    local**
07f75e8b5d57   minikube   bridge    local
56c131a7273d   none       null      local
```
you can now use kind to setup your HA local kubernetes cluster. 

Hope you enjoy.

`GOOD LABS !!!`
