# Manage Scheduling


## Scheduling Process

This section explains the scheduling procees in K8s, it describes that process taken to a assigned Pods to a node.

### Understanding Scheduling

kube-scheduler will find a node to schedule new Pods. Node are selected according to requirement that may been set based on:
- Resource requirements
- AFfinity and anti-affinity
- Taints and tolerations and more

Scheduler first finds feasible nodes and score them, node that selected is the highest score. Once node was found, the scheduler notifies the API server in a process called binding.

### Scheduler to Kubelet

Once scheduler decided, kubelet will pick up and instruct CRI to pull the image of the container on specific node. In the end, container is created and started.


## Node Preferences

Node preference allow us to run a pod on specific node using a label. A feild called `nodeSelecteor` in the `pod.spec` allow us to define the label which is set on nodes that are eligible to run the Pod. 

Using `kubectl label nodes` command will set the label on a node. Then we define the field in the `pod.spec` to mat the Pod to the specific node. `nodeName` is part of `pod.spec` and can be also use as field too.

### Setting up Node Preference

Set a label on a node
`kubectl label nodes node01 disktype=nvme`

Check a label attach on a node
`kubectl get nodes --show-labels`

Example config to use `nodeSelector` in `pod.Spec`
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: sata
```

Apply the config and see the event - *1 node(s) didn't match Pod's node affinity/selector*
```
controlplane:~/cka$ kubectl describe pod nginx
<<snippet>>
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  12s   default-scheduler  0/2 nodes are available: 1 node(s) didn't match Pod's node affinity/selector, 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }. no new claims to deallocate, preemption: 0/2 nodes are available: 2 Preemption is not helpful for scheduling.
```

Fix the selector and reapply should bring up the pod.
```
controlplane:~/cka$ kubectl describe pod nginx
<<snippet>>
Node-Selectors:              distype=nvme
<<Snippet>>
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  10s   default-scheduler  Successfully assigned default/nginx to node01
  Normal  Pulling    9s    kubelet            Pulling image "nginx"
  Normal  Pulled     4s    kubelet            Successfully pulled image "nginx" in 4.796s (4.796s including waiting). Image size: 62870438 bytes.
  Normal  Created    4s    kubelet            Created container: nginx
  Normal  Started    4s    kubelet            Started container nginx
```

## Affinity and Anti-Affinity Rules

It is an advanced scheduler rules. There are few types for affinity as below:
1. Node Affinity
2. Pod Affinity
3. Anti Affinity

 `Node` affinity use as a constrain a Pod to run on a specific node. Function like `nodeSelector` able schedule your pod into a specific node. There are 2 different options of Node Affinity as below:
1. `requiredDuringSchedulingIgnoredDuringExecution` - The scheduler can't schedule the Pod unless the rule is met.
2. `preferredDuringSchedulingIgnoredDuringExecution` - The scheduler tries to find a node that meets the rule. If a matching node is not available, the scheduler still schedules the Pod. Weightage or priority feature available in this option.

 Inter-Pod or Pod Affinity use as a constrain to run a Pod on a specific node that matches a running Pod (the pod also has label attached). In other word, if I have a Pod-A run on a node in a Zone-C, then the new Pod will be run in any of Zone-C nodes. 
 
Anti-affinity on the other hand, will avoid to run a Pod on a specific node that matches a running Pod (the pod also has label attached). In other word, if I have a Pod-A run on a node in a Zone-C, then the new Pod will be run in any nodes other than Zone-C nodes. There are 2 different options similar to Node Affinity as below:
 1. `requiredDuringSchedulingIgnoredDuringExecution`
 2. `preferredDuringSchedulingIgnoredDuringExecution`

### Node Affinity - Options 1

Below is the `nodeAffnity` example where Pod assigned to node that is label with `colors` = `GREEN` Prior to that, the worker already labelled:
```
apiVersion: v1
kind: Pod
metadata:
  name: affinity-green
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: colors
            operator: In
            values:
            - GREEN      
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

Worker that has been labelled
```
ansible@CTRL-01:~$ kubectl get nodes -l colors=GREEN
NAME     STATUS   ROLES    AGE   VERSION
wrk-02   Ready    <none>   47h   v1.34.3
```

Validate the pod assignement
```
ansible@CTRL-01:~$ kubectl get pods -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
affinity-green   1/1     Running   0          38s   172.16.19.65   wrk-02   <none>           <none>
```

### Node affinity - Options 2

Below is the `nodeAffnity` example where Pod assigned to node that is label with `colors` = `PURPLE` with weight. Prior to that, none of the workers labelled with `PURPLE`:
```
apiVersion: v1
kind: Pod
metadata:
  name: affinity-weight
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: colors
            operator: In
            values:
            - RED
      - weight: 50
        preference:
          matchExpressions:
          - key: colors
            operator: In
            values:
            - BLUE    
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

Worker has been labelled:
```
ansible@CTRL-01:~$ kubectl get nodes -l colors=RED
NAME     STATUS   ROLES    AGE   VERSION
wrk-01   Ready    <none>   47h   v1.34.3
ansible@CTRL-01:~$ kubectl get nodes -l colors=BLUE
NAME     STATUS   ROLES    AGE   VERSION
wrk-03   Ready    <none>   47h   v1.34.3
```

Validate pod assignement and highest weight should be selected
```
ansible@CTRL-01:~$ kubectl get pods -o wide
NAME              READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
affinity-green    1/1     Running   0          11m   172.16.19.65   wrk-02   <none>           <none>
affinity-weight   1/1     Running   0          18s   172.16.108.1   wrk-03   <none>           <none>
```

### Inter-Pod Affinity

First we run a pod with a label - `frontend`
```
apiVersion: v1
kind: Pod
metadata:
  name: web-tier-fe
  labels:
    tier: frontend
spec:
  containers:
  - name: nginx-container
    image: nginx
```

Validate the Pod label is there
```
ansible@CTRL-01:~$ kubectl get pods web-tier-fe --show-labels -o wide
NAME          READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES   LABELS
web-tier-fe   1/1     Running   0          4m13s   172.16.89.193   wrk-01   <none>           <none>            tier=frontend
```

Then we set a label that group all the worker in a specific group, the label should be a well-known labels.
```
ansible@CTRL-01:~$ kubectl get nodes -l topology.kubernetes.io/zone=MY -o wide --show-labels
NAME     STATUS   ROLES    AGE    VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME    LABELS
wrk-01   Ready    <none>   2d3h   v1.34.3   192.168.101.21   <none>        Ubuntu 24.04.3 LTS   6.8.0-90-generic   containerd://2.2.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,colors=RED,dc=primary,kubernetes.io/arch=amd64,kubernetes.io/hostname=wrk-01,kubernetes.io/os=linux,topology.kubernetes.io/zone=MY
wrk-02   Ready    <none>   2d3h   v1.34.3   192.168.101.22   <none>        Ubuntu 24.04.3 LTS   6.8.0-90-generic   containerd://2.2.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,colors=GREEN,dc=primary,kubernetes.io/arch=amd64,kubernetes.io/hostname=wrk-02,kubernetes.io/os=linux,topology.kubernetes.io/zone=MY
wrk-03   Ready    <none>   2d3h   v1.34.3   192.168.101.23   <none>        Ubuntu 24.04.3 LTS   6.8.0-90-generic   containerd://2.2.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,colors=BLUE,dc=primary,kubernetes.io/arch=amd64,kubernetes.io/hostname=wrk-03,kubernetes.io/os=linux,topology.kubernetes.io/zone=MY
```

Below is the Inter-Pod Affinity example where the Pod will place in a zone node that has a Pod that has been labelled `frontend`:
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity-frontend
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: tier
            operator: In
            values:
            - frontend
        topologyKey: topology.kubernetes.io/zone
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

Then we should see that the pod will run on any node within the `zone` that run the earlier pod `web-tier-fe`
```
ansible@CTRL-01:~$ kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
affinity-green          1/1     Running   0          4h4m    172.16.19.65    wrk-02   <none>           <none>
affinity-weight         1/1     Running   0          3h53m   172.16.108.1    wrk-03   <none>           <none>
pod-affinity-frontend   1/1     Running   0          7s      172.16.19.69    wrk-02   <none>           <none>
web-tier-fe             1/1     Running   0          45m     172.16.89.193   wrk-01   <none>           <none>
```

If we change the affinity to other than `frontend` and the result should failed as below:
```
ansible@CTRL-01:~$ kubectl describe pods pod-affinity-frontend --show-events
<<snippet>>
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  31s   default-scheduler  0/6 nodes are available: 3 node(s) didn't match pod affinity rules, 3 node(s) had untolerated taint(s). no new claims to deallocate, preemption: 0/6 nodes are available: 6 Preemption is not helpful for scheduling.
```
