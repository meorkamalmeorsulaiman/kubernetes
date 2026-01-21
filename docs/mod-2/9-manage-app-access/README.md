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

