# Managing Cluster Node

This section is working on managing the K8s cluster

## Table of Content
1. [Analyze Cluster Node]()
2. [Manage Node Container using crictl]()
3. [Static Pod]()
4. [Manage Node State]()
5. [Manage Node Service]()

## Analyze Cluster Node

Analyzing the cluster node is process that analyze the node rather than the K8s itself. A mixed of commands that can be use as below:
```
kubectl describe node wrk01
sudo systemctl status kubelet
sudo ls -lrt /var/log
sudo journalctl -u containerd
sudo journalctl -u kubelet
```

If metrics-server was installed, then we can use it
```
kubectl top nodes
```

## Manage Node Container using **crictl**

Pods are started as containers on node. `crictl` is a generic tools to connect to container runtime. Before using the `crictl`, you must make sure that the tools connect to the containerd socket correctly by looking at the setting in `/etc/crictl.yaml` Below is the containerd socket
```
runtime-endpoint: "unix:///run/containerd/containerd.sock"
```

Use that command to check the setting:
```
grep runtime-endpoint /etc/crictl.yaml
```

If all set, use below command to check the container running on a node
```
sudo crictl ps
sudo crictl ps -a
sudo crictl pods
```

`crictl` is a `docker` like command, and we can use several similar command as below
```
sudo crictl image
```

## Static Pod

Kubelet systemd process is configured to run static Pods from `/etc/kubernetes/manifests` Static pod essential to start K8s core components. We can run any static pod by copying manifest file into the directory. Then restart kubelet so that it will pickup. Never run the static pod on control node. Take the example manifest from control node and transfer to the worker using this command below command
```
kubectl run test-static-pod --image=nginx --dry-run=client -o yaml > test-static-pod.yaml
scp test-static-pod.yaml wrk01:~/
```

Use that output to move the manifest file in the `/etc/kubernemanifest/` on the worker node then restart kubelet as below:
```
ls
sudo mv test-static-pod.yaml /etc/kubernetes/manifests
sudo systemctl restart kubelet
```

Once restarted you can check on the worker node using
```
sudo crictl ps
```

Or in the control node
```
kubectl get pods
```

## Manage Node states

Sometime you need to manage or bring out of service. Few options:
- Cordon -  to mark a node as unschedule
- Drain -  to mark a node as unscheduleable and remove all running Pods
- Uncordon - to normalize


To cordon a node use below command:
```
kubectl cordon wrk01
kubectl get nodes
kubectl describe nodes wrk01 | grep Taints
```

To normalize use
```
kubectl uncordon wrk01
```

## Manage Node service

As of now, the service that runs K8s are:
1. containerd
2. kubelet

So we need to make sure that these services working correctly. We can check the individual service using `systemctl` and `journalctl` to check any logs. To further check the running processes us `ps` as below
```
sudo systemctl status containerd
sudo systemctl status kubelet
sudo journalctl -u containerd
sudo journalctl -u kubelet
ps afx | grep containerd
ps afx | grep kubelet
```
