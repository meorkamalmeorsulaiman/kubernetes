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

Procedures for restoring etcd:
1. Stop kubernestes service by moving the yaml configs somewhere else from `/etc/kubernetes/manifesets/*` 
2. Wait till the static pod gone, this can be validate using `sudo crictl ps`
3. After that, rename the etcd directory to something else `/var/lib/etcd`
4. Then proceed to restore using `sudo etcdctl snapshot restore /tmp/backup --data-dir /var/lib/etcd`
5. Proceed to move back the pod files into `/etc/kubernetes/manifests/*` and validate using `sudo crictl ps` to check the pod started

Start by creating an app to see the restoration different
```
ansible@CTRL-01:~$ kubectl create deploy before --image=nginx --replicas=1
deployment.apps/before created
ansible@CTRL-01:~$ kubectl get pods
NAME                     READY   STATUS              RESTARTS   AGE
before-6959b8ff4-5c8rk   0/1     ContainerCreating   0          9s
staticpod-ctrl-01        1/1     Running             0          11m
ansible@CTRL-01:~$ kubectl get deploy
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
before   1/1     1            1           56s
```

Let's move the pod files to other directory 
```
ansible@CTRL-01:/etc/kubernetes/manifests$ ls
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml  staticpod.yml
ansible@CTRL-01:/etc/kubernetes/manifests$ sudo mv * ..
ansible@CTRL-01:/etc/kubernetes/manifests$ ls -a
.  ..
```

Validate that the pod know gone
```
ansible@CTRL-01:/etc/kubernetes/manifests$ kubectl get pods
The connection to the server 192.168.101.11:6443 was refused - did you specify the right host or port?
ansible@CTRL-01:/etc/kubernetes/manifests$ sudo crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD                                        NAMESPACE
a794d2c65a86e       5e785d005ccc1       38 seconds ago      Running             calico-kube-controllers   1                   6d9db8dbafa95       calico-kube-controllers-7498b9bb4c-4zzpx   kube-system
b504dea1ad458       1cf5f116067c6       5 hours ago         Running             coredns                   0                   61f5cf8655048       coredns-674b8bbfcf-wr5bb                   kube-system
cd1b28ccc4ff9       1cf5f116067c6       5 hours ago         Running             coredns                   0                   99e24afeb142a       coredns-674b8bbfcf-kxkpt                   kube-system
ca43b3bf9fe8f       08616d26b8e74       5 hours ago         Running             calico-node               0                   c2869200e2f60       calico-node-tmnqj                          kube-system
66827df1c115d       0929027b17fc3       5 hours ago         Running             kube-proxy                0                   5d9d6a638b7ef       kube-proxy-vtn52                           kube-system
```

Rename the etcd library
```
sudo mv /var/lib/etcd /var/lib/etcd-prev-1
```

Now, proceed to restore the backup with `sudo etcdctl snapshot restore /tmp/etcdbackup.db --data-dir /var/lib/etcd`
```
{"level":"info","ts":1765803811.3748116,"caller":"snapshot/v3_snapshot.go:306","msg":"restoring snapshot","path":"/tmp/etcdbackup.db","wal-dir":"/var/lib/etcd/member/wal","data-dir":"/var/lib/etcd","snap-dir":"/var/lib/etcd/member/snap"}
{"level":"info","ts":1765803811.4173858,"caller":"mvcc/kvstore.go:388","msg":"restored last compact revision","meta-bucket-name":"meta","meta-bucket-name-key":"finishedCompactRev","restored-compact-revision":22136}
{"level":"info","ts":1765803811.424328,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"cdf818194e3a8c32","local-member-id":"0","added-peer-id":"8e9e05c52164694d","added-peer-peer-urls":["http://localhost:2380"]}
{"level":"info","ts":1765803811.4304008,"caller":"snapshot/v3_snapshot.go:326","msg":"restored snapshot","path":"/tmp/etcdbackup.db","wal-dir":"/var/lib/etcd/member/wal","data-dir":"/var/lib/etcd","snap-dir":"/var/lib/etcd/member/snap"}
```

Check the library
```
ansible@CTRL-01:/etc/kubernetes/manifests$ sudo ls /var/lib/etcd
member
```

Revert the static pod files
```
ansible@CTRL-01:/etc/kubernetes/manifests$ sudo mv ../*.yaml .
ansible@CTRL-01:/etc/kubernetes/manifests$ ls
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

Then check the pods, and few pods up now
```
ansible@CTRL-01:/etc/kubernetes/manifests$ sudo crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD                                        NAMESPACE
c3c7dc1689217       f457f6fcd712a       33 seconds ago      Running             kube-scheduler            0                   b2bd26ca2c7dd       kube-scheduler-ctrl-01                     kube-system
dd832960313f8       021d1ceeffb11       33 seconds ago      Running             kube-apiserver            0                   ca84c6615ea3e       kube-apiserver-ctrl-01                     kube-system
c1d0dc6846a30       29c7cab9d8e68       33 seconds ago      Running             kube-controller-manager   0                   7687b3571ea48       kube-controller-manager-ctrl-01            kube-system
1e781f6a19886       8cb12dd0c3e42       33 seconds ago      Running             etcd                      0                   e837fd635fcde       etcd-ctrl-01                               kube-system
67a6cf5ec0962       5e785d005ccc1       39 seconds ago      Running             calico-kube-controllers   6                   6d9db8dbafa95       calico-kube-controllers-7498b9bb4c-4zzpx   kube-system
b504dea1ad458       1cf5f116067c6       5 hours ago         Running             coredns                   0                   61f5cf8655048       coredns-674b8bbfcf-wr5bb                   kube-system
cd1b28ccc4ff9       1cf5f116067c6       5 hours ago         Running             coredns                   0                   99e24afeb142a       coredns-674b8bbfcf-kxkpt                   kube-system
ca43b3bf9fe8f       08616d26b8e74       5 hours ago         Running             calico-node               0                   c2869200e2f60       calico-node-tmnqj                          kube-system
66827df1c115d       0929027b17fc3       5 hours ago         Running             kube-proxy                0                   5d9d6a638b7ef       kube-proxy-vtn52                           kube-system
```

Check pods and we should not see any static pod and deploy craeted earlier. This validate that our backup is indeed correct.
```
ansible@CTRL-01:/etc/kubernetes/manifests$ kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS        AGE
calico-kube-controllers-7498b9bb4c-4zzpx   1/1     Running   6 (2m51s ago)   4h42m
calico-node-g9rr5                          1/1     Running   0               4h39m
calico-node-tmnqj                          1/1     Running   0               4h42m
coredns-674b8bbfcf-kxkpt                   1/1     Running   0               4h47m
coredns-674b8bbfcf-wr5bb                   1/1     Running   0               4h47m
etcd-ctrl-01                               1/1     Running   0               4h47m
kube-apiserver-ctrl-01                     1/1     Running   0               4h47m
kube-controller-manager-ctrl-01            1/1     Running   0               4h47m
kube-proxy-7vlmf                           1/1     Running   0               4h39m
kube-proxy-vtn52                           1/1     Running   0               4h47m
kube-scheduler-ctrl-01                     1/1     Running   0               4h47m
metrics-server-6b77496796-lh5lp            1/1     Running   0               3h40m
ansible@CTRL-01:/etc/kubernetes/manifests$ kubectl get deploy
No resources found in default namespace.
```



