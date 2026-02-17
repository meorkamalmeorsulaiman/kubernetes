# Maintenance tasks

Steps for maintenance work within the cluster. This topic divided into several sections:

#### Table of Contents

- [Metrics Server for Performance Metrics](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#metric-server-for-performance-metrics)
- [Backing up the Etcd](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#backing-up-the-etcd)
- [Restoring Etcd](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#restoring-etcd-backup)
- [Upgrade Control Node](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#upgrade-control-node)
- [Upgrade Worker Node](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#upgrade-worker-node)
- [K8s HA Cluster](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#k8s-ha-cluster)

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

If running an HA setup, you can repeat same setup but the upgrade of kubeadm command should be `sudo kubeadm upgrade node`

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

HA Cluster is a K8s setup of control node for availability and redundancy. The Keapalived run the VIP and HAProxy load balance the port 6443 request to all the controlls node. 6443 is the kube-apiserver run on the control pod

### Setup Keepalived and HAProxy

Using the script and it will setup the keepalived and HAProxy. 1st update the script and config files accordingly as below
```
cd ~/cka
sed -i 's/192.168.29.10/192.168.101.10/g' keepalived.conf
sed -i 's/ens33/eth0/g' keepalived.conf
sed -i 's/192.168.29.10/192.168.101.10/g' check_apiserver.sh
```

Run the HA setup script
```
sudo bash setup-lb-ubuntu.sh
```

### Initializing K8s HA Cluster

Proceed to setup HA cluster with the following command run on 1st control node
```
sudo kubeadm init --control-plane-endpoint "[VIP]:6443" --upload-certs
```

Then install CNI plugin on the 1st control node
```
kubectl apply -f  https://docs.projectcalico.org/manifests/calico.yaml
```

Copy the control node join command after the cluster initialization on the 1st control node to the rest of the control node
```
  kubeadm join 192.168.101.10:6443 --token [TOKEN] --discovery-token-ca-cert-hash [CERT-HASH] --control-plane --certificate-key [CERT-KEY]
```

Worker node join command
```
kubeadm join [VIP]:6443 --token [TOKEN] --discovery-token-ca-cert-hash [CERT-HASH]
```

> **Notes:** This steps are not required in the CKA exam