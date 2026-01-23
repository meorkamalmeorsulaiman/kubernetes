# Manage Application Access

## Kubernetes Networking

### Basic Networking Components

Network communications:
1. Between containers implemented as IPC - inter process communication. Not real networking
2. Between Pods implemented by network plugins
3. Between Pods and Services implemented by Service resources
4. Between external users and Services implemented by Services with ingress or gateway API

Incoming traffic manage by ingress but now replaced by gateway API. Both cant run on the same machine.

### Network Plugins

Required to implement networking and forwarding traffic between Pods. Several types of plugin available, choosing one based of it's feature support. Network plugin will add resource into K8s cluster using Custom Resource Definitions (CRD)

#### Seeing Network Configuration

Check CRD available
```
kubectl get crd
```

Check Pods IP Pools configured by Calico
```
ansible@CTRL-01:~$ kubectl get ippools -A -o yaml | grep cidr
    cidr: 172.16.0.0/16
```

See the actual Pod IP - should match the calico configs
```
ansible@CTRL-01:~$ kubectl get pods -o wide | awk '{print $1,$6}'
NAME IP
test-nginx-5bdcdd6c7c-zhlg5 172.16.89.193
```

Check the cluster IP range - 10.96.0.0/12 This can be change during the cluster initialization
```
ansible@CTRL-01:~$ ps afx | grep service-cluster-ip-range
  96131 pts/0    S+     0:00              \_ grep --color=auto service-cluster-ip-range
   8939 ?        Ssl    6:11  \_ kube-apiserver --advertise-address=192.168.101.11 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

## Service to Access Application

### Services

It's a resource to provice access to Pods. Different types of Service can be configured:
1. ClusterIP - Expose internally and reachable within the cluster
2. NodePort - Port mapping to allow external access to the pod
3. Loadbalancer - Load balancing traffic to NodePort or Cluster-IP based Services
4. ExternalName - Service discovery like that maps DNS names

#### Setting up Services

Create a deployment
```
kubectl create deploy web-service --image=nginx --replicas=3
```

Check deployment - should be ready
```
ansible@CTRL-01:~$ kubectl get pods --selector app=web-service -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
web-service-5899545bbb-9q7g2   1/1     Running   0          71s   172.16.19.65    wrk-02   <none>           <none>
web-service-5899545bbb-cbgzf   1/1     Running   0          71s   172.16.108.1    wrk-03   <none>           <none>
web-service-5899545bbb-f7vnq   1/1     Running   0          71s   172.16.89.194   wrk-01   <none>           <none>
```

Create a service for `web-service` deployment to expose externally - `NodePort` using port 80
```
kubectl expose deploy web-service --type=NodePort --port=80
```

Check the service details
```
ansible@CTRL-01:~$ kubectl describe svc web-service
Name:                     web-service
Namespace:                default
Labels:                   app=web-service
Annotations:              <none>
Selector:                 app=web-service
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.100.14.215
IPs:                      10.100.14.215
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31839/TCP
Endpoints:                172.16.108.1:80,172.16.89.194:80,172.16.19.65:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```

Above details explained as below:
- `Port` is the port on the service
- `TargetPort` is the actual container port
- `NodePort` is the externally expose port
The flow from external should be `Nodeport` > `Port` > `TargetPort` > `Endpoints`

We can test the web access on any of the node IP address.
```
ansible@CTRL-01:~$ curl -sI http://192.168.101.21:31839
HTTP/1.1 200 OK
Server: nginx/1.29.4
Date: Wed, 21 Jan 2026 05:54:03 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 09 Dec 2025 18:28:10 GMT
Connection: keep-alive
ETag: "69386a3a-267"
Accept-Ranges: bytes
```

## Ingress Controller

### Understanding Ingress

Ingress is a service discovery that consist of two parts:
- Physical LB available on the external network
- An API resource that contains rules to to forward to Service resources.

Exposes HTTP and HTTPS routes from outside the cluster to Services within the cluster. Ingress use `selectorLabel` in Services to forward to Pod. Traffic routing controlled by rules defined on the Ingress resource. We must have ingress controller in order to get ingress to work.

### Working with Nginx Ingress Controller

It installed from helm, 1st install helm
```
wget https://get.helm.sh/helm-v4.1.0-linux-amd64.tar.gz
tar -xvf helm-v4.1.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/
helm -version
```

Proceed to install Nginx-Ingress Controller
```
helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace
```

Check the pod
```
ansible@CTRL-01:~$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-6c657c6487-55ptn   1/1     Running   0          42s
```

Proceed to create deployment
```
kubectl create deploy nginxsvc --image=nginx --port=80
```

Expose the deployment, the service type should be default as ingress will manage that.
```
kubectl expose deploy nginxsvc --port=80
```

Check the serivce
```
ansible@CTRL-01:~$ kubectl describe svc nginxsvc
Name:                     nginxsvc
Namespace:                default
Labels:                   app=nginxsvc
Annotations:              <none>
Selector:                 app=nginxsvc
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.111.179.154
IPs:                      10.111.179.154
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
Endpoints:                172.16.19.66:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

Create an ingress, the rule is `nginxsvc.info/*=nginxsvc:80`
```
kubectl create ing nginxsvc --class=nginx --rule=nginxsvc.info/*=nginxsvc:80
```

Configure port forward all incoming request to the service, use `CTRL+Z` followed by `bg`
```
ansible@CTRL-01:~$ kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
^Z
[1]+  Stopped                 kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
ansible@CTRL-01:~$ bg
[1]+ kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80 &
```

Add host file
```
ansible@CTRL-01:~$ cat /etc/hosts | grep svc
127.0.0.1 localhost nginxsvc.info
```

Test
```
ansible@CTRL-01:~$ curl -sI nginxsvc.info:8080
HTTP/1.1 200 OK
Date: Fri, 23 Jan 2026 13:49:14 GMT
Content-Type: text/html
Content-Length: 615
Connection: keep-alive
Last-Modified: Tue, 09 Dec 2025 18:28:10 GMT
ETag: "69386a3a-267"
Accept-Ranges: bytes
```

## Ingress

Ingress rule catch incoming traffic that matches a specific path and optional hostname and connects that to a Service and port. To create rule use `kubectl create ingress` command. Different paths can be defined on the same host. 

### IngressClass

In one cluster, different ingress controllet can host and each run their own configuration. Controllers can be included in an `IngressClass`. When creating rule, use the `--class` option to implement the role on a specific Ingress controller.

#### Working with Rule

This rule is to demonstrate virtual host. 1st create deployment and Serivce with NodePort
```
kubectl create deploy web-service --image=nginx --replicas=3
kubectl expose deploy web-service --type=NodePort --port=80
```

Create ingress
```
kubectl create ingress web-service-ingress --rule="/=web-service:80" --rule="/hello=newdep:8080"
```

Add hosts file
```
ansible@CTRL-01:~$ cat /etc/hosts | grep web-service
127.0.0.1 nginxsvc.info web-service.info
```

Validate the ingress and we can see wildcard in the hosts
```
ansible@CTRL-01:~$ kubectl get ingress
NAME                  CLASS    HOSTS           ADDRESS   PORTS   AGE
nginxsvc              nginx    nginxsvc.info             80      5h7m
web-service-ingress   <none>   *                         80      2m8s
```

See the details, the error on `/hello` path
```
ansible@CTRL-01:~$ kubectl describe ingress web-service-ingress
Name:             web-service-ingress
Labels:           <none>
Namespace:        default
Address:
Ingress Class:    <none>
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /        web-service:80 (172.16.19.67:80,172.16.89.194:80,172.16.108.2:80)
              /hello   newdep:8080 (<error: services "newdep" not found>)
Annotations:  <none>
Events:       <none>
```

We deploy for `newdep`
```
kubectl create deploy newdep --image=gcr.io/google-sample/hello-app:2.0
kubectl expose deploy newdep --port=8080
```

Validate the ingress again
```
ansible@CTRL-01:~$ kubectl describe ingress web-service-ingress
Name:             web-service-ingress
Labels:           <none>
Namespace:        default
Address:
Ingress Class:    <none>
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /        web-service:80 (172.16.19.67:80,172.16.89.194:80,172.16.108.2:80)
              /hello   newdep:8080 ()
Annotations:  <none>
Events:       <none>
```
