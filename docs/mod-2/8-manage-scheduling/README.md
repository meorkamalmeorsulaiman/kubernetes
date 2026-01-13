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
