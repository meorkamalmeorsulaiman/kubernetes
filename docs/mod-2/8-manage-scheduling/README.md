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

### Anti-Affinity

First we unable relabel the node to SG zone
```
ansible@CTRL-01:~$ kubectl label node wrk-03 topology.kubernetes.io/zone=SG --overwrite=true
node/wrk-03 labeled
ansible@CTRL-01:~$ kubectl get node wrk-03 --show-labels
NAME     STATUS   ROLES    AGE    VERSION   LABELS
wrk-03   Ready    <none>   2d3h   v1.34.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,colors=BLUE,dc=primary,kubernetes.io/arch=amd64,kubernetes.io/hostname=wrk-03,kubernetes.io/os=linux,topology.kubernetes.io/zone=SG
```

Then we run a Pod with a `backend` label on `wrk-03`
```
apiVersion: v1
kind: Pod
metadata:
  name: web-tier-be
  labels:
    tier: backend
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: colors
            operator: In
            values:
            - BLUE
  containers:
  - name: nginx-container
    image: nginx
```

Validate it run on SG nodes
```
ansible@CTRL-01:~$ kubectl get pods web-tier-be -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
web-tier-be   1/1     Running   0          14s   172.16.108.3   wrk-03   <none>           <none>
```

We run a pod with Anit-Affinity matches Pod `backend` in SG zone
```
apiVersion: v1
kind: Pod
metadata:
  name: anti-affinity-backend
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: tier
              operator: In
              values:
              - backend
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

It will run on `MY` zone as use preferred and we have `backend` pod in `SG` zone 
```
ansible@CTRL-01:~$ kubectl get pods anti-affinity-backend -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
anti-affinity-backend   1/1     Running   0          46s   172.16.89.200   wrk-01   <none>           <none>
```

## Taint and Toleration

Taint is the opposite of Node AFfinity, it allow a node to repel a set of pods. Tolerations are applied to pods. Tolerations allow the scheduler to schedule pods with matching taints

Below setting up Taint on a node with a taint key. `NoSchedule` mean that no pod is allowed to be scheduled on `node01` unless it has matching toleration: 
```
kubectl taint node node01 tolerate=yes:NoSchedule
```

Check the taint key value:
```
controlplane:~$ kubectl describe node node01 | grep -i taints
Taints:             tolerate=yes:NoSchedule
```

Now we run a pod without a toleration
```
apiVersion: v1
kind: Pod
metadata:
  name: without-toleration
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

The pod should not be able to run
```
controlplane:~$ kubectl get pod without-toleration 
NAME                 READY   STATUS    RESTARTS   AGE
without-toleration   0/1     Pending   0          62s
```

Now we run a pod with toleration
```
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "tolerate"
    operator: "Equal"
    value: "yes"
    effect: "NoSchedule"
```

Pod should be able to run
```
controlplane:~$ kubectl get pod with-toleration 
NAME              READY   STATUS    RESTARTS   AGE
with-toleration   1/1     Running   0          6s
```

Details about taint and toleration on: [Taint and Toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)

## Manage Quota

A resource quota, defined by a ResourceQuota object, provides constraints that limit aggregate resource consumption per namespace. A ResourceQuota can also limit the quantity of objects that can be created in a namespace by API kind, as well as the total amount of infrastructure resources that may be consumed by API objects found in that namespace.

### Setting up Quota

Create a namespace
```
kubectl create namespace limited
```

Set the quota for the new namespace, it a hard quota with min pod = 3 and cpu equal to 100 milicoreee and memory is 5000Mib apply to namespace limited
```
kubectl create quota limited-quota --hard pods=3,cpu=100m,memory=500Mi -n limited
```

Check the quota applied 
```
controlplane:~$ kubectl describe ns limited
Name:         limited
Labels:       kubernetes.io/metadata.name=limited
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:     limited-quota
  Resource  Used  Hard
  --------  ---   ---
  cpu       0     100m
  memory    0     500Mi
  pods      0     3

No LimitRange resource.
```

Check quota details
```
controlplane:~$ kubectl describe quota limited-quota -n limited
Name:       limited-quota
Namespace:  limited
Resource    Used  Hard
--------    ----  ----
cpu         0     100m
memory      0     500Mi
pods        0     3
```

### Testing out Quota

Create a deployment
```
kubectl create deployment nginx --image=nginx --replicas=3 -n limited
```

No pods created
```
controlplane:~$ kubectl get all -n limited
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   0/3     0            0           49s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-66686b6766   3         0         0       49s
```

Check the deployment events - the problem related to the deployment resource restriction set
```
controlplane:~$ kubectl describe rs -n limited nginx-66686b6766
<<Snippet>>
Events:
  Type     Reason        Age                  From                   Message
  ----     ------        ----                 ----                   -------
  Warning  FailedCreate  3m35s                replicaset-controller  Error creating: pods "nginx-66686b6766-tftkx" is forbidden: failed quota: limited-quota: must specify cpu for: nginx; memory for: nginx
  Warning  FailedCreate  3m35s                replicaset-controller  Error creating: pods "nginx-66686b6766-7k2pd" is forbidden: failed quota: limited-quota: must specify cpu for: nginx; memory for: nginx
  Warning  FailedCreate  3m35s                replicaset-controller  Error creating: pods "nginx-66686b6766-p8w5h" is forbidden: failed quota: limited-quota: must specify cpu for: nginx; memory for: nginx
  Warning  FailedCreate  3m35s                replicaset-controller  Error creating: pods "nginx-66686b6766-r5gp5" is forbidden: failed quota: limited-quota: must specify cpu for: nginx; memory for: nginx
  Warning  FailedCreate  3m35s                replicaset-controller  Error creating: pods "nginx-66686b6766-nd5t6" is forbidden: failed quota: limited-quota: must specify cpu for: nginx; memory for: nginx
  Warning  FailedCreate  3m35s                replicaset-controller  Error creating: pods "nginx-66686b6766-zb8qx" is forbidden: failed quota: limited-quota: must specify cpu for: nginx; memory for: nginx
  Warning  FailedCreate  3m35s                replicaset-controller  Error creating: pods "nginx-66686b6766-9szmm" is forbidden: failed quota: limited-quota: must specify cpu for: nginx; memory for: nginx
  Warning  FailedCreate  3m35s                replicaset-controller  Error creating: pods "nginx-66686b6766-nsgjx" is forbidden: failed quota: limited-quota: must specify cpu for: nginx; memory for: nginx
  Warning  FailedCreate  3m34s                replicaset-controller  Error creating: pods "nginx-66686b6766-klfgc" is forbidden: failed quota: limited-quota: must specify cpu for: nginx; memory for: nginx
  Warning  FailedCreate  51s (x7 over 3m33s)  replicaset-controller  (combined from similar events): Error creating: pods "nginx-66686b6766-sbbmt" is forbidden: failed quota: limited-quota: must specify cpu for: nginx; memory for: nginx
```

Let's set the resource on the deployment
```
controlplane:~$ kubectl set resources deploy nginx --requests cpu=100m,memory=5Mi --limits cpu=200m,memory=20Mi -n limited
deployment.apps/nginx resource requirements updated
```

Check the deployment - it failed with 1 pod created 
```
controlplane:~$ kubectl get all -n limited
NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-877c5d664-dx662   1/1     Running   0          10s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/3     1            1           68s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-66686b6766   2         0         0       68s
replicaset.apps/nginx-877c5d664    2         1         1       10s
```

We hit the cpu quota limited 

```
controlplane:~$ kubectl describe quota limited-quota -n limited
Name:       limited-quota
Namespace:  limited
Resource    Used  Hard
--------    ----  ----
cpu         100m  100m
memory      5Mi   500Mi
pods        1     3
```

Edit the cpu limit to 500m
```
kubectl edit quota limited-quota -n limited
```

Check quota spec
```
controlplane:~$ kubectl describe quota limited-quota -n limited
Name:       limited-quota
Namespace:  limited
Resource    Used  Hard
--------    ----  ----
cpu         100m  500m
memory      5Mi   500Mi
pods        1     3
```

Deleted and redeploy the deployment
```
kubectl delete deploy nginx -n limited
kubectl create deploy nginx --image=nginx --replicas=3 -n limited
kubectl set resources deploy nginx --requests cpu=100m,memory=5Mi --limits cpu=200m,memory=20Mi -n limited
```

Check the deployement status
```
controlplane:~$ kubectl get all -n limited
NAME                        READY   STATUS    RESTARTS     AGE
pod/nginx-877c5d664-267nq   1/1     Running   1 (5s ago)   14s
pod/nginx-877c5d664-lddqq   1/1     Running   0            10s
pod/nginx-877c5d664-qgfhb   1/1     Running   1 (8s ago)   16s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   3/3     3            3           20s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-66686b6766   0         0         0       20s
replicaset.apps/nginx-877c5d664    3         3         3       16s
```

## LimitRange

A LimitRange is a policy to constrain the resource allocations (limits and requests) that you can specify for each applicable object kind (such as Pod or PersistentVolumeClaim) in a namespace. Quota applies in the entire namespace. More about LimitRange on [K8s Docs - LimitRanges](https://kubernetes.io/docs/concepts/policy/limit-range/)

### Working with LimitRange

Create a namespace
```
kubectl create ns limit-range
```

Create the limit range spec then apply it within the namespace
```
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```

Validate the limitrage applied
```
controlplane:~/cka$ kubectl describe ns limit-range
Name:         limit-range
Labels:       kubernetes.io/metadata.name=limit-range
Annotations:  <none>
Status:       Active

No resource quota.

Resource Limits
 Type       Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---  ---  ---------------  -------------  -----------------------
 Container  memory    -    -    256Mi            512Mi          -
```

### Using pod with limitRange

Run a pod withint the namespace
```
kubectl run limit-pod --image=nginx -n limit-range
```

Validate pod applied with the limit
```
controlplane:~/cka$ kubectl describe pod limit-pod -n limit-range
<<Snippet>>
  limit-pod:
    Container ID:   containerd://461e999089362870cf60c29bd73d82c1fe3c3757b883111128ac80f1633f9f11
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:c881927c4077710ac4b1da63b83aa163937fb47457950c267d92f7e4dedf4aec
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 19 Jan 2026 07:52:07 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      memory:  512Mi
    Requests:
      memory:     256Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-5zdtf (ro
```
