# Creating a K8s cluster

In this topic we learn how what is required to deploy a K8s cluster. 

## Table of Content

1. [Setting up Host]()
2. [Installing Container Runtime and K8s Tools]()
3. [Initializing Cluster on Control Node]()
4. [Installing CNI Plugin on Control Node]()
5. [Join a Cluster for Worker Node]()
6. [Initializing Cluster with config file]()


## Setting up Host node for K8s

Below are the high-level steps to get a cluster up and running:

1. Setting up kernel modules
2. Install container runtime
3. Install K8s tools - kubeadm, kubectl and kubelet
4. Initialize cluster and configure the kube client - kubectl
5. Set up CNI plugin on control node
6. Add other nodes
7. Verify other nodes availability

> **Notes:** Step 1 - 3 are not part of the exam.

### Setting up kernel modules 

Enable kernel modules: 
```
sudo modprobe overlay
sudo modprobe br_netfilter
```

Creata file with the name `/etc/sysctl.d/99-kubernetes-cri.conf` that includes these lines and restart your server to activate.
```
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv6.ip_forward = 1
```

Apply sysctl params without reboot `sudo sysctl --system`

## Installing container runtime and K8s tools

User scrit from the lab lesson by Sander to install. The scrip inclue setting up the kernel module that previouly configured.
```
sudo apt install git
git clone https://github.com/sandervanvugt/cka.git
cd cka
sudo bash setup-container.sh
```

The script will install container runtime and runc. At the end, you can check `systemd` for `containerd`
```
sudo systemctl status containerd
runc --version
```

Proceed to install K8s tools
```
sudo bash setup-kubetools.sh
```

Once finished, K8s tools should work. We can proceed the same step on other node - controls or workers. During the exam, this steps are not required
```
kubeadm version
kubectl version
```

## Initializing cluster on Control Node

This process will initialize a cluster. This step will be only done on a control node only. Below is the command to initialize a cluster. 
````
sudo kubeadm init
````

Once completed, it will display the command to set the kube client - kubectl. Below is the example command:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

It also include the join command as below:
```
kubeadm join 192.168.101.11:6443 --token 8kxc6d.4pmpum1pxn980qoq \
	--discovery-token-ca-cert-hash sha256:c8c8a64c443d8345590fb7bbe6c07835bc3249737d811d745baa3cee7e11e730 
```

The join command can be display again using:
```
sudo kubeadm token create --print-join-command
```

Once the kube client has been setup, you can start using the cluster using `kubectl` command:
```
kubectl version
```

It should specify the server version. That indicate it is working. In case rollback or reset is needed, use below command to reset the cluster:
```
sudo kubeadm reset
sudo kubeadm init
```
>**Notes:** Please reconfigure the kube client after reinitialize. The reset command doesn't delete the CNI plugin which has to be manually deleted.



## Installing CNI Plugin on Control Node

The CNI required by a cluster to allow node to node communication. The are several CNI plugins available but with the lesson demo. It suggest to use Calico. The CNI plugin will create a several pods and these pod will be created on all nodes in the cluster after they joined. It is good practice to install the CNI plugin on the Control node during the initial phase of the cluster initialization. Then proceed with worker node joining the cluster. Below is the command to install the plugin using manifest.
```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```
or using the lesson guide
```
kubectl apply -f  https://docs.projectcalico.org/manifests/calico.yaml
```

It will essentially create several pod for communication between node in the cluster. The pod usually starts with **calico** while using below command to check:
```
kubectl get pods -n kube-system
```

## Joining K8s cluster on worker node

To join the worker on into the cluster use the join command by print it using below command on the control node:
```
sudo kubeadm token create --print-join-command
```

Use the printed command in the worker node
```
sudo kubeadm join 192.168.101.11:6443 --token j3gy2b.ufbvx448uvirgcxp --discovery-token-ca-cert-hash sha256:8820f76f83a1eef0a60656e739548136b7ab34d72eb1d643b58f6486ee5291d1 
```

Once joined, you can validate the node status in the control node and wait until the pod is ready
```
kubectl get nodes
```

## Setting up cluster using config file

Use `sudo kubeadm config print init-defaults > config.yaml` to write all configuration parameters to `config.yaml` Edit the `config.yaml` with all desired parameters
- Set a least the following parameters:
     - localAPIEndpoint.advertiseAddress
          - Valid IP address on which the apiserver is available
    - nodeRegistration.name
          - The name of the apiservernode
Next, use `sudo kubeadm init --config config.yaml` to use the `config.yaml` file while installing the cluster

## Lab Practice

Task 1:
- Deploy K8s standalone cluster using `kubeadm`

Task 2:
- Deploy K8s standalone cluster using config file

