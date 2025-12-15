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
```
ansible@CTRL-01:~/cka$ sudo systemctl status containerd
● containerd.service - containerd container runtime
     Loaded: loaded (/usr/lib/systemd/system/containerd.service; enabled; preset: enabled)
     Active: active (running) since Mon 2025-12-15 08:07:16 UTC; 14s ago
       Docs: https://containerd.io
    Process: 7299 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 7300 (containerd)
      Tasks: 10
     Memory: 13.2M (peak: 14.0M)
        CPU: 92ms
     CGroup: /system.slice/containerd.service
             └─7300 /usr/bin/containerd

Dec 15 08:07:16 CTRL-01 containerd[7300]: time="2025-12-15T08:07:16.775688578Z" level=info msg="Start subscribing containerd event"
Dec 15 08:07:16 CTRL-01 containerd[7300]: time="2025-12-15T08:07:16.775753996Z" level=info msg="Start recovering state"
Dec 15 08:07:16 CTRL-01 containerd[7300]: time="2025-12-15T08:07:16.775836015Z" level=info msg="Start event monitor"
Dec 15 08:07:16 CTRL-01 containerd[7300]: time="2025-12-15T08:07:16.775842412Z" level=info msg=serving... address=/run/containerd/container>
Dec 15 08:07:16 CTRL-01 containerd[7300]: time="2025-12-15T08:07:16.775873197Z" level=info msg="Start snapshots syncer"
Dec 15 08:07:16 CTRL-01 containerd[7300]: time="2025-12-15T08:07:16.776204872Z" level=info msg=serving... address=/run/containerd/container>
Dec 15 08:07:16 CTRL-01 containerd[7300]: time="2025-12-15T08:07:16.776297171Z" level=info msg="containerd successfully booted in 0.027253s"
Dec 15 08:07:16 CTRL-01 containerd[7300]: time="2025-12-15T08:07:16.776296189Z" level=info msg="Start cni network conf syncer for default"
Dec 15 08:07:16 CTRL-01 containerd[7300]: time="2025-12-15T08:07:16.776323609Z" level=info msg="Start streaming server"
Dec 15 08:07:16 CTRL-01 systemd[1]: Started containerd.service - containerd container runtime.
```

