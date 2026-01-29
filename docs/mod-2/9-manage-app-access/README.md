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
kubectl create ingress web-service --class=nginx --rule="/=web-service:80" --rule="/hello=newdep:8080"
```

Validate the ingress and we can see wildcard in the hosts
```
ansible@CTRL-01:~$ kubectl get ingress
NAME          CLASS   HOSTS           ADDRESS   PORTS   AGE
nginxsvc      nginx   nginxsvc.info             80      7h36m
web-service   nginx   *                         80      8m48s
```

See the details, the error on `/hello` path
```
ansible@CTRL-01:~$ kubectl describe ingress web-service
Name:             web-service
Labels:           <none>
Namespace:        default
Address:
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /        web-service:80 (172.16.89.195:80,172.16.19.68:80,172.16.108.4:80)
              /hello   newdep:8080 ()
Annotations:  <none>
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  Sync    40s   nginx-ingress-controller  Scheduled for sync
```

We deploy for `newdep`
```
kubectl create deploy newdep --image=gcr.io/google-sample/hello-app:2.0
kubectl expose deploy newdep --port=8080
```

Validate the ingress again
```
ansible@CTRL-01:~$ kubectl describe ingress web-service
Name:             web-service
Labels:           <none>
Namespace:        default
Address:
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /        web-service:80 (172.16.89.195:80,172.16.19.68:80,172.16.108.4:80)
              /hello   newdep:8080 (172.16.108.5:8080)
Annotations:  <none>
Events:
  Type    Reason  Age    From                      Message
  ----    ------  ----   ----                      -------
  Normal  Sync    4m37s  nginx-ingress-controller  Scheduled for sync
```

We can access these Pod by using the ingress-controller IP. Below is the ingress controller IP by checking the ingress controller service. This is below we are not exporing the nodeport for the ingress-controller service during installation with helm
```
ansible@CTRL-01:~$ kubectl get service -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.98.168.34    <pending>     80:30283/TCP,443:30668/TCP   7h39m
ingress-nginx-controller-admission   ClusterIP      10.104.168.73   <none>        443/TCP                      7h39m
```

Test default the ingress using `10.98.168.34`
```
ansible@CTRL-01:~$ curl -H "Host: web-service.info" http://10.98.168.34
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Test path
```
ansible@CTRL-01:~$ curl -H "Host: web-service.info" http://10.98.168.34/hello
Hello, world!
Version: 2.0.0
Hostname: newdep-76896cd768-s4st6
```

## Port Forwarding

Use to connect apps for analyzing and troubleshooting. Foward traffic coming in to a local port on the kubectl client machine to a port that is available in a Pod. Example `kubectl port-forward mypod 5555:80` - forward local port 5555 to Pod `mypod` port 80 Then, `ctrl+z` or a `&` at the end of the command to run in the background. 

## Gateway API

### Understanding Gateway API

Replacement for Ingress, added more features. Use several resources:
- GatewayClass - represents the Gateway controller
- Gateway: defaines an instance of traffice handling infra
- HTTPRoute: how traffic is routed

We need to setup Gateway API controller in order to get it to working. Gateway API controllers are provided by the ecosystem. Example Nginx Gateway Fabric. Before installing the controller, you must install the custom resource

#### Resource - GatewayClass

Represent physical Gateway Controller. It uses`spec.controllerName` to connect to a specific Gateway Controller.

#### Resource - Gateway

Multiple Gateways can connect to one Gateway Controller, at least one Gateway is required. The Gateway uses the `gatewayClassName` property to connect to the `GatewayController`. It also defines `listeners` to specify which protocols should be serviced.

#### Resource - HTTPRoutes

Defines to which Service an incoming request should be forwarded. Incoming requests are identified by the `spec.hostnames`. The `parentRefs` property connects the HTTPRoute to a Gateway. The `backendRefs` property connects the HTTPRoute to a Service.

## Using Gateway API to Access Application

High-level steps to provision:
1. Make sure custom resources available
2. Install a community Gateway API - not in vanilla K8s
3. Verify the community Gateway Controller is ready
4. Create K8s application
5. Configure GatewayClass, Gateway and HTTPRoute
6. Test by accessing the Service that exposes the community controller

### Setup Gateway API

Install custom resources, this will installed the custom resource required - release version should refer to K8s sigs [Gateway API](https://github.com/kubernetes-sigs/gateway-api)
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
```

Validate CRDs installed 
```
kubectl get crds | grep gateway.network
```

Install community gateway controller - nginx-gateway-fabric with nodePort type [Helm Artifact](https://artifacthub.io/packages/helm/nginx-gateway-fabric/nginx-gateway-fabric/1.5.1)
```
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --version 1.5.1  --create-namespace -n nginx-gateway --set nginx.service.type=NodePort
```

Or install without NodePort options and later edit
```
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric --version 1.5.1  --create-namespace -n nginx-gateway
```

Validate helm installed
```
helm list --all-namespaces
```

Edit service type
```
kubectl edit -n nginx-gateway svc ngf-nginx-gateway-fabric

<<Snippet>>
    type: NodePort
  status:
    loadBalancer: {}
<<Snippet>>
```

Validate deployment
```
ansible@CTRL01:~$ kubectl get all -n nginx-gateway
NAME                                           READY   STATUS    RESTARTS   AGE
pod/ngf-nginx-gateway-fabric-5bff9d865-mgpsv   1/1     Running   0          104s

NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/ngf-nginx-gateway-fabric   ClusterIP   10.106.179.170   <none>        443/TCP   104s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ngf-nginx-gateway-fabric   1/1     1            1           104s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ngf-nginx-gateway-fabric-5bff9d865   1         1         1       104s
```

Validate the gateway controller - gc
```
ansible@CTRL01:~$  kubectl get gc
NAME    CONTROLLER                                   ACCEPTED   AGE
nginx   gateway.nginx.org/nginx-gateway-controller   True       2m7s
```

Validate gateway service set to nodePort
```
ansible@CTRL01:~$ kubectl get svc -n nginx-gateway
NAME                       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ngf-nginx-gateway-fabric   NodePort   10.96.114.70   <none>        80:30612/TCP,443:31956/TCP   114s
```

Create K8s resource - deployment and service
```
kubectl create deploy webservice --image=nginx --replicas=3
kubectl expose deploy webservice --port=80
```

Create the gatewayClassName, listener, parentRefs, hostnames, and rules
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: webservice-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: webservice-route
spec:
  parentRefs:
  - name: webservice-gateway
  hostnames:
  - "webservice.com"
  rules:
  - backendRefs:
    - name: webservice
      port: 80
```

Apply the config and check HTTPRoute
```  
ansible@CTRL01:~$ kubectl get httproute
NAME               HOSTNAMES            AGE
webservice-route   ["webservice.com"]   22s
```

Test using gateway api cluster IP - positive
```
ansible@CTRL01:~$ curl -H "Host: webservice.com" http://10.96.114.70
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Test using gateway api cluster IP - false positive
```
ansible@CTRL01:~$ curl -H "Host: webservice.org" http://10.96.114.70
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```
