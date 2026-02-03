# Practice Lab - Exam 1

## Create Cluster

- Deploy a cluster with version 1.34.1 with one control node.
- Join a worker node.

## Static Pod

- Create one static pod using nginx image

## Metrics Server

- Install metrics server 

## Backup and Restore etcd

- Deploy a virutal machine `vm01` with ubuntu image and ensure ssh install
- Expose ssh service for `vm01` with port 32022
- Backup etcd by creating a snapshot
- Delete the ssh service for `vm01`
- Restore etcd, ensure that the previous state restored

## Upgrades control and worker node

- Update control and worker to latest v1.34
### Control Node
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

### Worker Node
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