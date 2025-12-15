# Maintenance tasks

Steps for maintenance work within the cluster

## Metric servers for performance metrics

Not part of vanilla, have to install separately. Repo can be accessible from here: `https://github.com/kubernetes-sigs/metrics-server` Below is to install the metric server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

We should see the new pod, but it's not ready
```
ansible@CTRL-01:~$ kubectl get pods -n kube-system | grep metrics
metrics-server-867d48dc9c-r5kn7            0/1     Running   0          14m
ansible@CTRL-01:~$ kubectl logs -n kube-system metrics-server-867d48dc9c-r5kn7 | tail -n 1
E1215 09:24:05.336642       1 scraper.go:149] "Failed to scrape node" err="Get \"https://192.168.101.21:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 192.168.101.21 because it doesn't contain any IP SANs" node="wrk-01"
```

We need to update the pod so that it allow insecure TLS using `kubectl edit -n kube-system deployments.apps metrics-server` by adding an argument
```
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --kubelet-insecure-tls
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        image: registry.k8s.io/metrics-server/metrics-server:v0.8.0
```

Wait until the new pod come up
```
ansible@CTRL-01:~$ kubectl get pods -n kube-system | grep metrics
metrics-server-6b77496796-lh5lp            1/1     Running   0          49s
```

Now we can use the metric server 
```
ansible@CTRL-01:~$ kubectl top node
NAME      CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)   
ctrl-01   203m         5%       758Mi           4%          
wrk-01    55m          1%       339Mi           2%
```

Other useful command `kubectl top pod -A`


##  Backing up the Etcd

Etc started as static pod, run the kube-system stuff. We need `etcdctl` tool to backup the etcd. Use `sudo apt install etcd-client` to install etcd tool.
```
Output
```

- In older `etcdctl` use wrong API vesion by default, has to fix by using `sudo ETCDCTL_API=3 etcdctl ... snapshot save`
- To use `etcdctl` we need to spcify the etcd API endpoint including the cacert, cert, and key to be used
- The values can be found using `ps aux | grep etcd` or inside `/etc/kubernetes/pki/etcd/`
```
ansible@CTRL-01:~$ sudo ps aux | grep etcd
root        8811  4.0  1.9 1529148 317388 ?      Ssl  08:18   9:58 kube-apiserver --advertise-address=192.168.101.11 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
root        8820  2.1  0.4 11739772 69560 ?      Ssl  08:18   5:20 etcd --advertise-client-urls=https://192.168.101.11:2379 --cert-file=/etc/kubernetes/pki/etcd/server.crt --client-cert-auth=true --data-dir=/var/lib/etcd --experimental-initial-corrupt-check=true --experimental-watch-progress-notify-interval=5s --initial-advertise-peer-urls=https://192.168.101.11:2380 --initial-cluster=ctrl-01=https://192.168.101.11:2380 --key-file=/etc/kubernetes/pki/etcd/server.key --listen-client-urls=https://127.0.0.1:2379,https://192.168.101.11:2379 --listen-metrics-urls=http://127.0.0.1:2381 --listen-peer-urls=https://192.168.101.11:2380 --name=ctrl-01 --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt --peer-client-cert-auth=true --peer-key-file=/etc/kubernetes/pki/etcd/peer.key --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt --snapshot-count=10000 --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
ansible   157554  0.0  0.0   3956  2048 pts/0    S+   12:25   0:00 grep --color=auto etcd
```

Before backing up, test the api endpoint using the following command:
```
sudo etcdctl --endpoints=192.168.101.11:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key get / --prefix --keys-only
```

Once working, proceed to do the actual backup
```
sudo etcdctl --endpoints=192.168.101.11:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key snapshot save /tmp/etcdbackup.db
```

File should be created
```
ansible@CTRL-01:~$ ls /tmp/etcdbackup.db 
/tmp/etcdbackup.db
```

Test the backup file
```
ansible@CTRL-01:~$ sudo etcdctl --write-out=table snapshot status /tmp/etcdbackup.db 
+---------+----------+------------+------------+
|  HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+---------+----------+------------+------------+
| c5b6b9f |    22743 |        958 |     3.8 MB |
+---------+----------+------------+------------+
```

Before proceed to restore, let's create a static pod. This can help to differentiate the backup process. Create the pod file
```
ansible@CTRL-01:~$ sudo cat /etc/kubernetes/manifests/staticpod.yml 
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: staticpod
  name: staticpod
spec:
  containers:
  - image: nginx
    name: staticpod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Validate that the pod running
```
ansible@CTRL-01:~$ kubectl get pod
sNAME                READY   STATUS    RESTARTS   AGE
staticpod-ctrl-01   1/1     Running   0          35s
```

## Restoring Etcd backup


