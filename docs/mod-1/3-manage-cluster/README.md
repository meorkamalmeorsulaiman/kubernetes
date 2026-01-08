# Managing Cluster Node

This section is working on managing the K8s cluster

## Analyze cluster node

Several commands can be use as below:
- Use `sudo systemctl status kubelet` to get runtime information about the kubelet
- Use log file in `/var/log` as well as `journalctl` output to get access to logs
- Generic node information is obtained through `kubectl describe [node name]`
- If the Metrics Server is installed, use `kubectl top nodes` to get a summary of CPU/memory usage on a node

## Using `crictl` to manage node containers

- All Pods are started as containers on node
- `crictl` is a generic tools to connect to container runtime
- To use it, a runtime-endpoint and image-endpoint need to be set
- The config in `/etc/crictl.yaml` file

Using `ccrictl` to check pod
```
ansible@CTRL-01:~$ sudo crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD                                        NAMESPACE
5221309fb8654       5e785d005ccc1       15 minutes ago      Running             calico-kube-controllers   0                   6d9db8dbafa95       calico-kube-controllers-7498b9bb4c-4zzpx   kube-system
b504dea1ad458       1cf5f116067c6       15 minutes ago      Running             coredns                   0                   61f5cf8655048       coredns-674b8bbfcf-wr5bb                   kube-system
cd1b28ccc4ff9       1cf5f116067c6       15 minutes ago      Running             coredns                   0                   99e24afeb142a       coredns-674b8bbfcf-kxkpt                   kube-system
ca43b3bf9fe8f       08616d26b8e74       16 minutes ago      Running             calico-node               0                   c2869200e2f60       calico-node-tmnqj                          kube-system
66827df1c115d       0929027b17fc3       21 minutes ago      Running             kube-proxy                0                   5d9d6a638b7ef       kube-proxy-vtn52                           kube-system
4f5a266453f91       8cb12dd0c3e42       21 minutes ago      Running             etcd                      0                   8a98ed2485995       etcd-ctrl-01                               kube-system
e5b203e8df1f4       29c7cab9d8e68       21 minutes ago      Running             kube-controller-manager   0                   2eecb86fa413f       kube-controller-manager-ctrl-01            kube-system
9b394a4694596       021d1ceeffb11       21 minutes ago      Running             kube-apiserver            0                   bc8b77f370a3f       kube-apiserver-ctrl-01                     kube-system
17a0dfda51e2f       f457f6fcd712a       21 minutes ago      Running             kube-scheduler            0                   2830dd6deb9e8       kube-scheduler-ctrl-01                     kube-system
```

Few commands available to use:
- Check pod that has been scheduled `sudo crictl pods`
- Check pod images `sudo crictl images`

## Static pods

Kubelet systemd process is configured to run static Pods from `/etc/kubernetes/manifests` Static pod essential to start K8s core components. We can run any static pod by copying manifest file into the directory. Then restart `kubelet` so that it will pickup. Never run this on contorl node. Take the example manifest from control node using this command `kubectl run test-staticpod --image=nginx --dry-run=client -o yaml` Copy the output and use it in the worker node:
```
ansible@WRK-01:~$ sudo ls /etc/kubernetes/manifests/ -l
total 4
-rw-rw-r-- 1 ansible ansible 352 Jan  6 14:02 static.yaml
```

Kubelet should pickup right away
```
ansible@CTRL-01:~$ kubectl get pods -o wide
NAME                    READY   STATUS              RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
test-staticpod-wrk-01   0/1     ContainerCreating   0          18s   <none>   wrk-01   <none>           <none>
```

## Node states

Sometime you need to manage or bring out of service. Few options:
- `kubectl cordon` to mark a node as unschedule
- `kubectl drain` to mark a node as unscheduleable and remove all running Pods
- `kubectl uncordon` to normalize

Cordon a node of `wrk-01`
```
ansible@CTRL-01:~$ kubectl cordon wrk-01
node/wrk-01 cordoned
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS                     ROLES           AGE   VERSION
ctrl-01   Ready                      control-plane   29m   v1.33.7
wrk-01    Ready,SchedulingDisabled   <none>          21m   v1.33.7
ansible@CTRL-01:~$ kubectl describe node wrk-01 | grep -i taint
Taints:             node.kubernetes.io/unschedulable:NoSchedule
```

Normalize
```
ansible@CTRL-01:~$ kubectl uncordon wrk-01
node/wrk-01 uncordoned
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS   ROLES           AGE   VERSION
ctrl-01   Ready    control-plane   30m   v1.33.7
wrk-01    Ready    <none>          22m   v1.33.7
ansible@CTRL-01:~$ kubectl describe node wrk-01 | grep -i taint
Taints:             <none>
```

## Node service

## Lab Practice 

Tasks:
- Create and run static pod

