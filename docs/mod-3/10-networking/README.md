# Networking

## Manage CNI and Network Plugins

### Understanding CNI

CNI is a standard for container network. CNI use to configure network interface in Linux container along with several supported plugins. CNI Plugins use to implement the standard in container networking. Details about [CNI](https://github.com/containernetworking/cni) 

## Service Auto Registration and K8s DNS

### Understanding Service Auto Registration

K8s runs coredns Pods in the kube-system Namespace as internal DNS servers. These Pods are exposed by the `kube-dns` Service, which default offers access on IP Address 10.96.0.10 (clusterIP) K8s Service resources register with this kubedns service. Below is the service
```
ansible@CTRL01:~$ kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   10h
```

Pods are automatically configured with the IP address of the kubedns Service as their DNS resolver. As a result, all Pods can access all Services by name

### Using Service by Name in same namespace

Create a pod
```
kubeclt run webserver --image=nginx
```

Create service to expose the webserver port 80
```
kubectl expose pod webserver --port=80
```

Create a machine that can access the webserver 
```
kubectl run jumppod --image=busybox -- sleep 3600
```

Test connecting to the webserver using names
```
ansible@CTRL01:~$ kubectl exec -it jumppod -- wget webserver
Connecting to webserver (10.102.70.69:80)
saving to 'index.html'
index.html           100% |***************************************************************|   615  0:00:00 ETA
'index.html' saved
```

> **Notes:** To attach to the shell use `kubectl exec --stdin --tty jumppod -- /bin/sh`

### Using Service by Name in different namespace

If a Service is running in the same Namespace, it can be reached from a Pod by using its short hostname. If a Service is running in another Namespace, an FQDN consisting of `servicename.namespace.svc.clustername` must be used. The clustername is defined in the coredns ConfigMap Corefile definition and set to `cluster.local` if it hasn't been changed, use `kubectl get cm -n kube-system coredns -o yaml` to verify. Below is working out Service that run on different namespace.

Create another pod in different namespace
```
kubectl create ns prod-vrf
kubectl run prod-jumppod --image=busybox -n prod-vrf -- sleep 3600
```

Check the DNS settings for the new pod in `prod-vrf`
```
kubectl exec -it prod-jumppod -n prod-vrf -- cat /etc/resolv.conf
```

Try to lookup previously create pod in the default namespace. It should fail
```
kubectl exec -it prod-jumppod -n prod-vrf -- nslookup webserver
```

Try to lookup previously created pod in the default namespace using the FQDN format
```
kubectl exec -it prod-jumppod -n prod-vrf -- nslookup webserver.default.svc.cluster.local
```

## Network Policies

No restrictions between traffic in K8s even in other namespaces. NetworkPolicies can be use to limit communication. CNI Plugin must first support this feature. In NetworkPlicies, traffic will be denined if there is no match. Identifiers used to apply the network policy. There are 3 identifiers:
- Pods - podSelector
- Namespaces - namespaceSelector
- IP blocks - ipBlock

A selector label is used to specify what traffic is allowed to and from the Pods that match the selector

### Manage Traffic between Pods

Let's create 2 pod
```
kubectl run webserver --image=nginx
kubectl expose pod webserver --port=80
kubectl run jump01 --image=ubuntu -- sleep 3600
```

Let's test connection from the jump host to the webserver
```
ansible@CTRL01:~$ kubectl exec -it jump01 -- curl http://webserver -I
HTTP/1.1 200 OK
Server: nginx/1.29.4
Date: Sat, 31 Jan 2026 02:13:46 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 09 Dec 2025 18:28:10 GMT
Connection: keep-alive
ETag: "69386a3a-267"
Accept-Ranges: bytes
```

Let's setup the networkPolicy
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: http-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: webserver
  ingress:
  - from:
    - podSelector:
        matchLabels:
          http: "true"
```

Above describe:
- The policy apply on all pod in default namespace that has label `run=webserver`
- The ingress apply to any pod in default namespace with label `http=true` will be allowed to connect

Let's apply the policy test connection from `jump01` It should fail
```
root@jump01:/# curl http://webserver -I --connect-timeout 1
curl: (28) Failed to connect to webserver port 80 after 1002 ms: Timeout was reached
```

Let's attach label to `jump0` and test again
```
ansible@CTRL01:~$ kubectl label pod jump01 http=true
pod/jump01 labeled
ansible@CTRL01:~$ kubectl exec -it jump01 -- curl http://webserver -I --connect-timeout 1
HTTP/1.1 200 OK
Server: nginx/1.29.4
Date: Sat, 31 Jan 2026 02:31:07 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 09 Dec 2025 18:28:10 GMT
Connection: keep-alive
ETag: "69386a3a-267"
Accept-Ranges: bytes
```

networkPolicy will apply the rule within the namespace where it was created. If no matching label in the ingress/egress podSelector, the ingress and egress will be apply to all pod within the name space. Below is the example:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: http-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      run = webserver
  ingress:
  - from:
    - podSelector: {}
```

### Manage traffic between namespaces

To allow traffic from different namespace, we need to unspecified the ingress namespaceSelector. Let's start by creating new jump host in a new namespace
```
kubectl create ns vrf-green
kubectl run jump03 -n vrf-green --image=ubuntu -- sleep 3600
```

Let's test the connection to the webserver
```
root@jump03:/# curl -I http://webserver.default.svc.cluster.local --connect-timeout 1
curl: (28) Failed to connect to webserver.default.svc.cluster.local port 80 after 1002 ms: Timeout was reached
```

Let's update the networkpolicy so that it access from any namespace
```
ansible@CTRL01:~$ cat netPolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: http-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: webserver
  ingress:
  - from:
    - namespaceSelector: {}
    - podSelector:
        matchLabels:
          http: "true"
```

Let's test again
```
ansible@CTRL01:~$ kubectl exec -n vrf-green --stdin --tty jump03 -- /bin/bash
root@jump03:/# curl -I http://webserver.default.svc.cluster.local --connect-timeout 1
HTTP/1.1 200 OK
Server: nginx/1.29.4
Date: Sat, 31 Jan 2026 03:01:41 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 09 Dec 2025 18:28:10 GMT
Connection: keep-alive
ETag: "69386a3a-267"
Accept-Ranges: bytes
```

`jump03` doesn't have label but get's permitted. This is because the ingress rule is in `OR` condition. In order to get and `AND` condition. The hyphen must be remove from ingress `podSelector` Let update the policy as below:
```
ansible@CTRL01:~$ cat netPolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: http-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: webserver
  ingress:
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          http: "true"
```

Let's test and we should expect the `jump03` failed
```
ansible@CTRL01:~$ kubectl exec -it -n vrf-green jump03 -- curl -I http://webserver.default.svc.cluster.local --connect-timeout 1
curl: (28) Failed to connect to webserver.default.svc.cluster.local port 80 after 1002 ms: Timeout was reached
command terminated with exit code 28
ansible@CTRL01:~$ kubectl exec -it jump01 -- curl http://webserver -I --connect-timeout 1
HTTP/1.1 200 OK
Server: nginx/1.29.4
Date: Sat, 31 Jan 2026 03:08:56 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 09 Dec 2025 18:28:10 GMT
Connection: keep-alive
ETag: "69386a3a-267"
Accept-Ranges: bytes
```

Now attach the label to `jump03` and test it
```
ansible@CTRL01:~$ kubectl label pod jump03 http=true -n vrf-green
pod/jump03 labeled
ansible@CTRL01:~$ kubectl exec -it -n vrf-green jump03 -- curl -I http://webserver.default.svc.cluster.local --connect-timeout 1
HTTP/1.1 200 OK
Server: nginx/1.29.4
Date: Sat, 31 Jan 2026 03:09:42 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 09 Dec 2025 18:28:10 GMT
Connection: keep-alive
ETag: "69386a3a-267"
Accept-Ranges: bytes
```

Much more granular rules can go to [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

## Lab

Run a web service in restriced namesapce and expose it. A jump host reside in default namespace where `jump01` allow SSH to `jump02` While `jump02` is the only pod that can access the web service in the restricted namespace.
