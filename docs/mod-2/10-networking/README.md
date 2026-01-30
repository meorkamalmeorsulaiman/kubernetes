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



