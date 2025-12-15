# Creating a K8s cluster

Requirements:
- Require kubeadm
- At least 2 nodes
- Container runtime and kubernetes tools must installed to proceed with cluster installation with kubeadm

Installation procedures:
1. Setting up kernel modules
2. Install container runtime
3. Install K8s tools - kubeadm, kubectl and kubelet
4. Initialize cluster
5. Configure the kube client - kubectl
6. Set up CNI plugin
7. Add other nodes
8. Verify other nodes availability

## Setting up kernel modules

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

User scrip from the demo to install
```
apt install git
git clone
cd cka
sudo ./setup-container.sh
```

The script actually do few stuff:
1. Setup kernel
2. Downloading containerd and runc binary and install

At the end, you can check `systemd` for `containerd`
