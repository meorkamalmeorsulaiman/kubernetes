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

## Affinity and anti-Affinity Rules

It a advanced scheduler rules. `Node` affinity use as a constrain for a node to run a Pod by matching the labels attached to the nodes. `Node` affnity function like `nodeSelector` but able to put more advance rule. `Inter-Pod` affinity constrains nodes to run a Pod by matching the labels of existing Pods already running on that node. `Anti-affinity` can only applied between Pods so that are not run together.

A Pod that has a `nodeAffinity` label will only assigned to nodes that matches the label. A Pod that has a `podAffinity` label will only assigned to nodes that run a Pods that matching label.

2 different options that can be use to define node affinity:
1. `requiredDuringSchedulingIgnoredDuringExecution` - requires the node to meet the constraint that is defined.
2. `preferredDuringSchedulingIgnoredDuringExecution` - defines a soft affinity that is ignored it it cannot be fulfilled, priority can be assigned to define priorities - weight

Affinity is applied while sheduling Pods and cant be change on Pod that has been running.

To define affinity label, a `matchExporession` is used to define a `key`, an `operator` as well as optionally one or more values. 

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

