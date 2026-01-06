# Creating a K8s cluster

Requirements:
- Require kubeadm
- At least 2 nodes
- Container runtime and kubernetes tools must installed to proceed with cluster installation with kubeadm

Creating K8s Cluster:
1. Setting up kernel modules
2. Install container runtime
3. Install K8s tools - kubeadm, kubectl and kubelet
4. Initialize cluster
5. Configure the kube client - kubectl
6. Set up CNI plugin
7. Add other nodes
8. Verify other nodes availability

> :memo: **Note:** Step 1 - 3 aren't part of the exam.

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

Proceed to install K8s tools
```
sudo ./setup-kubetools.sh
```

At the end, `kubectl` should work
```
ansible@CTRL-01:~/cka$ kubectl version
Client Version: v1.33.7
Kustomize Version: v5.6.0
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

## Initializing cluster

- Initialize a K8 cluster using kubeadm on the control node only
- With root privileges, use `sudo kubeadm init` to start initialize
- At the end, you should see the `kubeclt` config steps:
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

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.101.11:6443 --token rm6lw5.inu6zii4zpnr0huw \`
```

Configure `kubectl` as below:
```
mkdir ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $USER ~/.kube/config
```

At the end you should able to use `kubectl`
```
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS     ROLES           AGE    VERSION
ctrl-01   NotReady   control-plane   2m6s   v1.33.7
```

## Setting up CNI plugin

Setup this on control node (control-node only), use below command to install calico
```
kubectl apply -f  https://docs.projectcalico.org/manifests/calico.yaml
```

At the end, you will see new pod setup:
```
ansible@CTRL-01:~$ kubectl get pod -n kube-system
NAME                                       READY   STATUS              RESTARTS   AGE
calico-kube-controllers-7498b9bb4c-4zzpx   0/1     ContainerCreating   0          30s
calico-node-tmnqj                          0/1     Running             0          30s
coredns-674b8bbfcf-kxkpt                   0/1     ContainerCreating   0          5m17s
coredns-674b8bbfcf-wr5bb                   0/1     ContainerCreating   0          5m17s
etcd-ctrl-01                               1/1     Running             0          5m23s
kube-apiserver-ctrl-01                     1/1     Running             0          5m23s
kube-controller-manager-ctrl-01            1/1     Running             0          5m23s
kube-proxy-vtn52                           1/1     Running             0          5m17s
kube-scheduler-ctrl-01                     1/1     Running             0          5m24s
```

Validate that all the pod running
```
ansible@CTRL-01:~$ kubectl get pod -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7498b9bb4c-4zzpx   1/1     Running   0          64s
calico-node-tmnqj                          1/1     Running   0          64s
coredns-674b8bbfcf-kxkpt                   1/1     Running   0          5m51s
coredns-674b8bbfcf-wr5bb                   1/1     Running   0          5m51s
etcd-ctrl-01                               1/1     Running   0          5m57s
kube-apiserver-ctrl-01                     1/1     Running   0          5m57s
kube-controller-manager-ctrl-01            1/1     Running   0          5m57s
kube-proxy-vtn52                           1/1     Running   0          5m51s
kube-scheduler-ctrl-01                     1/1     Running   0          5m58s
```

## Joining K8s cluster on worker node

Now, we have initialize and setup the cluster on the controller node. We proceed with joining the cluster using below command on the worker node:
```
sudo kubeadm join 192.168.101.11:6443 --token ol6rkf.jrvbp3hol590yygm --discovery-token-ca-cert-hash sha256:7e03bf0fcd6ed447213f3220bfb64fef9a1986412b91faee1c6e5800d29aef5a
```

At the end, you should see in the controller node that the worker listed as part of the cluster node:
```
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS     ROLES           AGE     VERSION
ctrl-01   Ready      control-plane   9m13s   v1.33.7
wrk-01    NotReady   <none>          40s     v1.33.7
```

Validate the node is ready
```
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS   ROLES           AGE     VERSION
ctrl-01   Ready    control-plane   9m54s   v1.33.7
wrk-01    Ready    <none>          81s     v1.33.7
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

