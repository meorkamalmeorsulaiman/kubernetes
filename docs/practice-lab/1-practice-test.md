# Practice Lab - Exam 1

## Create Cluster

### Deploy K8s Cluster
- Deploy a cluster with version 1.34.1 with one control node.
- Join a worker node.

### Deploy K8s HA Cluster
- Use the scipts to install CRI, kubetools and load-balancer setup
- Configure HA cluster with 3 control plane and 2 worker nodes
- Ensure that the control plane node can be use as a client as well
```
#On all nodes
sudo apt install git 
git clone https://github.com/sandervanvugt/cka.git
cd cka
sudo sh setup-container-previous-version.sh
sudo sh setup-kubetools-previousversion.sh 

#On master node
cd cka
sed -i 's/192.168.29.100/192.168.101.10/g' keepalived.conf
sed -i 's/192.168.29.100/192.168.101.10/g' check_apiserver.sh
sed -i 's/ens33/eth0/g' keepalived.conf
sh setup-lb-ubuntu.sh
ip 

#Initialize cluster and install CNI plugin on master node
sudo kubeadm init --control-plane-endpoint "192.168.101.10:6443" --upload-certs

#Setup kubectl and install CNI Plugin on master node
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f  https://docs.projectcalico.org/manifests/calico.yaml

#Join cluster for control node
sudo kubeadm join 192.168.101.10:6443 --token ne88rc.i0hyvcxpsmsywv34 \
--discovery-token-ca-cert-hash sha256:0e8c8f67d56d1b2703d12c07127a173a28f9d2d3314cdeb5a21c6533d60c435c \
--control-plane --certificate-key 51ae26989dec706c6ab9e1d4f9048269ce2583d04526134dbba159f00528978f

#Join cluster for worker node
sudo kubeadm join 192.168.101.10:6443 --token ne88rc.i0hyvcxpsmsywv34 \
--discovery-token-ca-cert-hash sha256:0e8c8f67d56d1b2703d12c07127a173a28f9d2d3314cdeb5a21c6533d60c435c
```

## Manage Cluster

### Static Pod

- Create one static pod using nginx image
```
#Generate spec file
kubectl run static-pod --image=nginx --dry-run=client -o yaml > static-pod.yaml

#Move the spec file to k8s manifest location 
sudo mv static-pod.yaml /etc/kubernetes/manifests/

#Validate static pod
kubectl get pods
```

### Drain
- Drain a worker node in the cluster

## Maintenance Task

### Metrics Server

- Install metrics server
```
#Deploy metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.8.1/components.yaml

#Edit deployment so that it will ignore tls - kubelet-ignore-tls
kubectl edit deploy -n kube-system metrics-server
```

### Backup and Restore etcd
- Deploy a virutal machine `vm01` with ubuntu image and ensure ssh install
- Expose ssh service for `vm01` with port 32022
```
kubectl expose pod vm01 --type=NodePort --port=32022 --target-port=22
```

- Backup etcd by creating a snapshot
```
#Install etcdctl and etcdutl binaries
wget https://github.com/etcd-io/etcd/releases/download/v3.6.7/etcd-v3.6.7-linux-amd64.tar.gz
tar xzf etcd-v3.6.7-linux-amd64.tar.gz -C /tmp/etcd-binaries --strip-components=1 --no-same-owner
cd /tmp/etcd-binaries/
sudo mv etcdctl /usr/bin
sudo mv etcdutl /usr/bin
etcdctl version
etcdutl version

#Create snapshot for backup
sudo etcdctl --endpoints=localhost:2379 \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key \
snapshot save /tmp/etcdbackup.db
```

- Delete the ssh service for `vm01`
```
kubectl delete svc vm01
```

- Restore etcd, ensure that the previous state restored
```
#Stop etcd and kube-apiserver pod
cd /etc/kubernetes/manifest
sudo mv *.yaml ..
sudo mv /var/lib/etcd /var/lib/etcd-backup01

#Ensure that the pods stop
sudo crictl ps

#Proceed to load the backup
sudo etcdutl snapshot restore /tmp/etcdbackup.db --data-dir /var/lib/etcd
cd /etc/kubernetes/manifest
sudo mv ../*.yaml .

#Validate that the both etcd and kube-apiserver are up 
sudo crictl ps
kubectl get nodes
```

### Upgrades control and worker node
- Update control and worker to latest v1.34

#### Control Node
```
#Update repo to new version
sudo sed -i 's/v1.34/v1.35/g' /etc/apt/sources.list.d/kubernetes.list 
sudo apt update
sudo apt-cache madison kubeadm

#Set the latest version for kubeadm package
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.35.0-*' && \
sudo apt-mark hold kubeadm

#Verify that download package updated
kubeadm version

#Precheck the ugprade
sudo kubeadm upgrade plan

#Proceed to upgrade if no issue
sudo kubeadm upgrade apply 1.35.0

#Drain the control node before proceeding to upgrade kubelet and kubectl
kubectl drain ctrl01 --ignore-daemonsets

#Proceed to upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl && sudo apt update && \
sudo apt install -y kubelet='1.35.0-*' kubectl='1.35.0-*' && \
sudo apt-mark hold kubelet kubectl

#Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

#Normalize control node
kubectl uncordon ctrl01

#Verfiy the version upgraded
kubectl get nodes
```

#### Worker Node
```
#Update repo to new version
sudo sed -i 's/v1.34/v1.35/g' /etc/apt/sources.list.d/kubernetes.list 
sudo apt update
sudo apt-cache madison kubeadm

#Set the latest version for kubeadm package
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.35.0-*' && \
sudo apt-mark hold kubeadm

#Verify that download package updated
kubeadm version

#Proceed to upgrade 
sudo kubeadm upgrade node

#Drain the worker node before proceeding to upgrade kubelet and kubectl from control node
kubectl drain wrk01 --ignore-daemonsets

#Proceed to upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl && sudo apt update && \
sudo apt install -y kubelet='1.35.0-*' kubectl='1.35.0-*' && \
sudo apt-mark hold kubelet kubectl

#Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

#Validate worker node version on control node
kubectl get nodes
kubectl uncordon wrk01
```

## Security

### RBAC

## Deploy Application

### Provision a Pod using Deployment
- Start a deployment consists of:
  - Web service running nginx in default namespace
  - Replica should be 3
  - Service run on 32080 for http connection

### DeamonSet

### StatefulSet

### InitContainer

### Scale Application

### Horizontal Pod AutoScaler

### SideCar

## Templating

- Setup helm
- Install HashiCorp Vault using Helm

## Scheduling

### Node Preference

- Deploy an application and an sure that it will deploy on a specific node using `nodeSelector`

### Node Affinity

- Affinity setup - name `COLORS`:
  - `wrk01` - GREEN
  - `wrk02` - BLUE
  - `wrk03` - RED
- Deploy an app where it will use BLUE color and wont able to run if the rule aren't met.

### Inter-Pod Affinity

### Anti-Affinity

### Quota

## App Access

### Service

- Deploy a jump server running ubuntu and setup ssh with user lab
- Expose ssh to port 32022 that is accessible from Node IP

### Ingress Controller

### Gateway API

- Run a deployment name cka-web with 3 nginx replicas.
- Expose the webservice and accessible using FQDN - cka.mmkmss.com

## Networking

### Service Name
- Create a webservice on port 80 in default namespace
- Expose the service using port 32080
- Create a `jumpbox02` in prod namesapce and `jumpbox03` in dev namespace running ubuntu and install curl
- Test access to webservie using name

### Network Policies
- Create a webservice in dev and prod namespace
- Ensure that the the webservice can be accessible from the same jumpbox namespace only

## Storage

- Create a PersistentVolume using HostPath storage type to access the directory `/storage`
- Create an index.thml inside `/storage`
- Run a webservice pod with nginx and mount the HostPath on to `/usr/share/nginx/html`
- Expose the pod to validate the webservice hompage.