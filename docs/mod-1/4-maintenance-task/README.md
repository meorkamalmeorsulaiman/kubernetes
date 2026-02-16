# Maintenance tasks

Steps for maintenance work within the cluster. This topic divided into several sections:

#### Table of Contents

- [Metrics Server for Performance Metrics](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#metric-servers-for-performance-metrics)
- [Backing up Etcd](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#backing-up-the-etcd)
- [Restoring Etcd](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#backing-up-the-etcd)
- [Upgrade Control Node](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#backing-up-the-etcd)
- [Upgrade Worker Node](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#upgrade-worker-node)
- [K8s HA Cluster](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#k8s-ha-cluster)
- [K8s HA Cluster Configuration](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#k8s-ha-cluster-configuration)
- [Demo K8s HA Cluster Deployment](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#demo-k8s-ha-cluster-deployment)

## Metric Server for Performance Metrics

Metrics-server allow us to get k8s performance metrics. Repo can be accessible from here: [Metrics Server SiGS](https://github.com/kubernetes-sigs/metrics-server) Below is to install the metric server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Edit the deployment pod parameter so that it will ignore tls. Otherwise, it wont work
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

Once the pod come up. Then can use the metric server by checking the available commands:
```
kubectl top -h
kubectl top node
kubectl top pod
```

##  Backing up the Etcd

Below are the backup steps:
1. Ensure `etcdctl` and `etcdutl` binaries installed
2. Take a backup snapshot using `etcdctl`

To install the binaries, which can be download and extract from [etcd-github](https://github.com/etcd-io/etcd) Once download, extract the binary. The steps included in the download page. Below is the basic
```
wget https://github.com/etcd-io/etcd/releases/download/v3.6.7/etcd-v3.6.7-linux-amd64.tar.gz
mkdir /tmp/etcd
tar xzvf etcd-v3.6.7-linux-amd64.tar.gz -C /tmp/etcd --strip-components=1 --no-same-owner
sudo cp /tmp/etcd/etcdctl /usr/bin
sudo cp /tmp/etcd/etcdutl /usr/bin
etcdctl version
etcdutl version
```

In older `etcdctl` use wrong API vesion by default, has to fix by using 
```
sudo ETCDCTL_API=3 etcdctl ... snapshot save
```

To use backup etcd we need to spcify the etcd API endpoint including the cacert, cert, and key to be used. The values can be found using 
```
ps aux | grep etcd
```

Or it is actually located in
```
/etc/kubernetes/pki/etcd/
```

Before backing up, test the api endpoint using the following command:
```
sudo etcdctl --endpoints=[IP]:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key get / --prefix --keys-only
```

Once working, proceed to do the actual backup
```
sudo etcdctl --endpoints=localhost:2379 --cacert [Location] --cert [server cert] --key [server key] snapshot save /tmp/etcdbackup.db
```

Test the backup file
```
sudo etcdutl snapshot status /tmp/etcdbackup.db  --write-out=table
```

## Restoring Etcd backup

Procedures for restoring etcd:
1. Stop kubernestes service by moving the yaml configs somewhere else from `/etc/kubernetes/manifesets/*` 
2. Wait till the static pod gone, this can be validate using `sudo crictl ps`
3. After that, rename the etcd directory to something else `/var/lib/etcd`
4. Then proceed to restore using `sudo etcdctl snapshot restore /tmp/backup --data-dir /var/lib/etcd`
5. Proceed to move back the pod files into `/etc/kubernetes/manifests/*` and validate using `sudo crictl ps` to check the pod started


Let's move the pod files to other directory 
```
cd /etc/kubernetes/manifests
sudo mv *.yaml ..
sudo mv /var/lib/etcd /var/lib/etcd-backup01
```

Validate that the pod for `etcd` and `kube-apiserver` gone
```
sudo crictl ps
```

Once, proceed to restore the backup with
```
sudo etcdutl snapshot restore /tmp/etcdbackup.db --data-dir /var/lib/etcd
```

Revert the static pod files
```
cd /etc/kubernetes/manifests/
sudo mv ../*.yaml .
ls
```

Then check the pods, etcd and kube-apiserver should be running and stable
```
sudo crictl ps
```

Check pods and svc which should be restored
```
kubectl get all
```

## Upgrade control node

The process of upgrading the k8s require several steps but on high-level. The steps is actually to upgrade the kubetools and resinitialize the cluster. Below are the steps:
1. Update the repo, install and mark the version to the latest for kubetools
2. Upgrade the kubeadm
3. Cordon the node and upgrade the kubelet and kubectl


### Update Repo

Start by updating the repo version to desired version (current is v1.34). Ref in https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/
```
sudo sed -i 's/v1.34/v1.35/g' /etc/apt/sources.list.d/kubernetes.list 
```

Get the updated version
```
sudo apt update
sudo apt-cache madison kubeadm
```

### Upgrade kubeadm

Proceed to upgrade using
```
sudo apt-mark unhold kubeadm && sudo apt-get update && sudo apt-get install -y kubeadm='1.35.1-*' && sudo apt-mark hold kubeadm
```

Check kubeadm should be upgraded version - v1.35
```
kubeadm version
```

Proceed to check cluster upgrade and available version using 
```
sudo kubeadm upgrade plan
```

Then apply the upgrade 
```
sudo kubeadm upgrade apply v1.35.1 
```

### Cordon node and upgrade kubelet and kubectl

Proceed to drain the control node for kubelet and kubectl upgrade
```
kubectl drain control --ignore-daemonsets
```

Upgrade kubelet and kubectl using 
```
sudo apt-mark unhold kubelet kubectl && sudo apt-get update && \
sudo apt-get install -y kubelet='1.35.1-*' kubectl='1.35.1-*' && \
sudo apt-mark hold kubelet kubectl
```

Then restart the service
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Next, uncorden the control node and validate the version installed
```
kubectl uncordon ctrl01
kubectl get nodes
```

## Upgrade worker node

The process are similar to the control node, but using different command
1. Update the repo, install and mark the version to the latest for kubetools
2. Upgrade the kubeadm
3. Cordon the node and upgrade the kubelet and kubectl

### Upgrade kubeadm

Proceed to upgrade using
```
sudo apt-mark unhold kubeadm && sudo apt-get update && sudo apt-get install -y kubeadm='1.35.1-*' && sudo apt-mark hold kubeadm
```

Check kubeadm should be upgraded version - v1.35
```
kubeadm version
```

Upgrade kubeadm using 
```
sudo kubeadm upgrade node
```

### Cordon the worker node using the controller

```
kubectl drain wrk01 --ignore-daemonsets
```

### Upgrade kubelet and kubectl

Proceed to upgrade kubelet and kubectl using
```
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.35.1-*' kubectl='1.35.1-*' && \
sudo apt-mark hold kubelet kubectl
```

Then restart the service
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Next, uncorden the control node and validate the version installed
```
kubectl uncordon wrk01
kubectl get nodes
```


## K8s HA Cluster

### HA Options
HA Options:
1. Run the control plane and etcd on the same node - stacked control plane
2. Seperate the control plane and etcd on different nod - External etcd cluster

### HA Requirements
HA Requirments:
1. Load balancing with LB to distribute workload between the control nodes
2. LB feature can use external software
3. In the exam, LB setup isn't required

### K8s HA Cluster Configuration
Configuration:
1. Use HAProxy to load-balance port 6443 across the control nodes
2. Traffic forwarded to `kube-apiserver` port 6443
3. Use keepalived on all control node with a VIP
4. `kubectl` connect to the VIP instead of the `CTRL-01` IP address
5. Use provided script from the repo `setup-lb-ubuntu.sh` to setup the HA - run on one of the control node only

#### Keepalived and HAProxy Setup
Setting up the `keepalived` and `HAProxy` by running the script prior to initializing the cluster, update the VIP and the interface name accordingly. At the end you should be able to check the VIP
```
ansible@CTRL-01:~/cka$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether bc:24:11:fc:6f:e6 brd ff:ff:ff:ff:ff:ff
    altname enp0s18
    inet 192.168.101.11/24 brd 192.168.101.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 192.168.101.10/24 scope global secondary eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::be24:11ff:fefc:6fe6/64 scope link 
       valid_lft forever preferred_lft forever
```

#### Initializing K8s HA Cluster
Proceed to setup HA cluster with the following steps:
1. Run 
```
sudo kubeadm init --control-plane-endpoint "[VIP]:6443" --upload-certs
```
2. Configure the `kubectl` client
3. Setup CNI plugin with
```
kubectl apply -f  https://docs.projectcalico.org/manifests/calico.yaml
```
4. Copy and use the `kubeadm join` command, there will be 2 join commands - control and worker. Use option `--control-plane` for control node

Control node join command
```
  kubeadm join 192.168.101.10:6443 --token [TOKEN] --discovery-token-ca-cert-hash [CERT-HASH] --control-plane --certificate-key [CERT-KEY]
```

Worker node join command
```
kubeadm join [VIP]:6443 --token [TOKEN] --discovery-token-ca-cert-hash [CERT-HASH]
```

5. Verify the cluster nodes using `kubectl get nodes`


### Demo K8s HA Cluster Deployment

Setup the HA Cluster by starting on `CTRL-01` and should see the client setup and 2 join commands
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes running the following command on each as root:

  kubeadm join 192.168.101.10:6443 --token wgax9h.xsu2183yrdpon934 \
        --discovery-token-ca-cert-hash sha256:ce0d6e0ab6b99715b37054e0c67acc1767059bae9533ea8438168aa56a0be733 \
        --control-plane --certificate-key 77433bfa22546aad2bb784fbc201d3573a6726b375abf8f87ca37f1603d39745

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.101.10:6443 --token wgax9h.xsu2183yrdpon934 \
        --discovery-token-ca-cert-hash sha256:ce0d6e0ab6b99715b37054e0c67acc1767059bae9533ea8438168aa56a0be733 
```

Proceed to configure the `kubectl`
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

After that, proceed to install the CNI plugin
```
kubectl apply -f  https://docs.projectcalico.org/manifests/calico.yaml
```

Moving on, join the remaining control nodes into the cluster. Setup the kube client too. 
```
ansible@CTRL-02:~$ sudo kubeadm join 192.168.101.10:6443 --token wgax9h.xsu2183yrdpon934 \
        --discovery-token-ca-cert-hash sha256:ce0d6e0ab6b99715b37054e0c67acc1767059bae9533ea8438168aa56a0be733 \
        --control-plane --certificate-key 77433bfa22546aad2bb784fbc201d3573a6726b375abf8f87ca37f1603d39745
## Snippet ##
To start administering your cluster from this node, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

Validate the cluster nodes and check all controllers that
```
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS   ROLES           AGE     VERSION
ctrl-01   Ready    control-plane   7m18s   v1.34.3
ctrl-02   Ready    control-plane   3m4s    v1.34.3
ctrl-03   Ready    control-plane   26s     v1.34.3
```

Proceed to join the worker node in the cluster using the worker join command
```
ansible@WRK-01:~/cka$ sudo kubeadm join 192.168.101.10:6443 --token wgax9h.xsu2183yrdpon934 \
        --discovery-token-ca-cert-hash sha256:ce0d6e0ab6b99715b37054e0c67acc1767059bae9533ea8438168aa56a0be733 
[preflight] Running pre-flight checks
[preflight] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[preflight] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 1.001429479s
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

Validate the cluster nodes
```
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS   ROLES           AGE     VERSION
ctrl-01   Ready    control-plane   11m     v1.34.3
ctrl-02   Ready    control-plane   6m50s   v1.34.3
ctrl-03   Ready    control-plane   4m12s   v1.34.3
wrk-01    Ready    <none>          74s     v1.34.3
```

Test by shutting down `ctrl-01` for testing and login to `ctrl-02`
```
ansible@ANS-CTRL:~/kubernetes$ ssh 192.168.101.10
ansible@CTRL-02:~$ kubectl get nodes
NAME      STATUS     ROLES           AGE     VERSION
ctrl-01   NotReady   control-plane   13m     v1.34.3
ctrl-02   Ready      control-plane   9m39s   v1.34.3
ctrl-03   Ready      control-plane   7m1s    v1.34.3
wrk-01    Ready      <none>          4m3s    v1.34.3
ansible@CTRL-02:~$ kubectl get pod -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-b45f49df6-sl42r   1/1     Running   0          10m
calico-node-4jkgm                         1/1     Running   0          6m6s
calico-node-8lxfh                         1/1     Running   0          10m
calico-node-gng5l                         1/1     Running   0          3m8s
calico-node-nrdw9                         1/1     Running   0          8m44s
coredns-66bc5c9577-hdp2c                  1/1     Running   0          12m
coredns-66bc5c9577-wjhnc                  1/1     Running   0          12m
etcd-ctrl-01                              1/1     Running   0          12m
etcd-ctrl-02                              1/1     Running   0          8m44s
etcd-ctrl-03                              1/1     Running   0          6m3s
kube-apiserver-ctrl-01                    1/1     Running   0          12m
kube-apiserver-ctrl-02                    1/1     Running   0          8m44s
kube-apiserver-ctrl-03                    1/1     Running   0          6m3s
kube-controller-manager-ctrl-01           1/1     Running   0          12m
kube-controller-manager-ctrl-02           1/1     Running   0          8m44s
kube-controller-manager-ctrl-03           1/1     Running   0          6m3s
kube-proxy-2nwmt                          1/1     Running   0          8m44s
kube-proxy-4rlwr                          1/1     Running   0          6m6s
kube-proxy-6kgj2                          1/1     Running   0          12m
kube-proxy-vpnkt                          1/1     Running   0          3m8s
kube-scheduler-ctrl-01                    1/1     Running   0          12m
kube-scheduler-ctrl-02                    1/1     Running   0          8m44s
kube-scheduler-ctrl-03                    1/1     Running   0          6m3s
ansible@CTRL-02:~$ 
```

## Lab Practice

Tasks:
- Backup Etcd and restore Restore Etcd

