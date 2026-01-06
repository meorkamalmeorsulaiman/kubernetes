# Deploying Kubernetes Application

### Topics
1. [Using Deployment](#URL)
2. [Running Agents with DeamonSets](#URL)
3. [Using StatefulSets](#URL)
4. [Individual Pod](#URL)
5. [Manage Pod initialization](#URL)
6. [Scaling application](#URL)
7. [Autoscale](#URL)
8. [Sidecar for logging](#URL)


## Using Deployments

Standard to run k8s deployment. Deployments responsible to start the pod. Deployments resources use replicaset to scale pod. Deployment offer rolling update feature. To deploy use `kubectl create deploy` 
```
ansible@CTRL-01:~$ kubectl create deploy mondeploy --image=nginx:1:17 --replicas=3
deployment.apps/mondeploy created
ansible@CTRL-01:~$ kubectl get all --selector app=mondeploy
NAME                             READY   STATUS             RESTARTS   AGE
pod/mondeploy-746c94dc94-269gc   0/1     InvalidImageName   0          15s
pod/mondeploy-746c94dc94-2xntz   0/1     InvalidImageName   0          15s
pod/mondeploy-746c94dc94-jppzh   0/1     InvalidImageName   0          15s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mondeploy   0/3     3            0           15s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/mondeploy-746c94dc94   3         3         0       15s
```

You can scale the deployment as below:
```
ansible@CTRL-01:~$ kubectl scale deployment mondeploy --replicas=4
deployment.apps/mondeploy scaled
ansible@CTRL-01:~$ kubectl get all --selector app=mondeploy
NAME                             READY   STATUS             RESTARTS   AGE
pod/mondeploy-746c94dc94-269gc   0/1     InvalidImageName   0          81s
pod/mondeploy-746c94dc94-2xntz   0/1     InvalidImageName   0          81s
pod/mondeploy-746c94dc94-jppzh   0/1     InvalidImageName   0          81s
pod/mondeploy-746c94dc94-lt44g   0/1     InvalidImageName   0          2s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mondeploy   0/4     4            0           81s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/mondeploy-746c94dc94   4         4         0       81s
```

Update deployment image - rolling update deployment:
```
ansible@CTRL-01:~$ kubectl set image deploy mondeploy nginx=nginx:latest
deployment.apps/mondeploy image updated
ansible@CTRL-01:~$ kubectl get all
NAME                             READY   STATUS              RESTARTS   AGE
pod/mondeploy-746c94dc94-269gc   0/1     InvalidImageName    0          2m30s
pod/mondeploy-746c94dc94-2xntz   0/1     InvalidImageName    0          2m30s
pod/mondeploy-746c94dc94-jppzh   0/1     InvalidImageName    0          2m30s
pod/mondeploy-85b887cb9d-8z8n4   0/1     ContainerCreating   0          8s
pod/mondeploy-85b887cb9d-kjthl   0/1     ContainerCreating   0          8s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   69m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mondeploy   0/4     2            0           2m30s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/mondeploy-746c94dc94   3         3         0       2m30s
replicaset.apps/mondeploy-85b887cb9d   2         2         0       8s
ansible@CTRL-01:~$ kubectl get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/mondeploy-85b887cb9d-8z8n4   1/1     Running   0          48s
pod/mondeploy-85b887cb9d-kjthl   1/1     Running   0          48s
pod/mondeploy-85b887cb9d-p6xbf   1/1     Running   0          34s
pod/mondeploy-85b887cb9d-zjmw5   1/1     Running   0          34s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   70m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mondeploy   4/4     4            4           3m10s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/mondeploy-746c94dc94   0         0         0       3m10s
replicaset.apps/mondeploy-85b887cb9d   4         4         4       48s
```

## DeamonSets

A resource that starts application in each cluster node. It start necessary agents on all cluster nodes. We can use for user workloads. Convert deployment to DaemonSets by creating the config file `kubectl create deploy daemon --image=nginx --dry-run=client -o yaml > daemon.yml` Edit by changing `kind` to `DaemonSet`, remove `replicas` and `strategy`:
```
ansible@CTRL-01:~$ cat daemon.yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: daemon
  name: daemon
spec:
  selector:
    matchLabels:
      app: daemon
  template:
    metadata:
      labels:
        app: daemon
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

Next apply with `kubectl apply -f daemon.yml`
```
ansible@CTRL-01:~$ kubectl apply -f daemon.yml
daemonset.apps/daemon created
ansible@CTRL-01:~$ kubectl get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/daemon-bz6n6                 1/1     Running   0          4s
pod/daemon-llfdt                 1/1     Running   0          4s
pod/daemon-r7z7z                 1/1     Running   0          4s
pod/mondeploy-85b887cb9d-8z8n4   1/1     Running   0          7m13s
pod/mondeploy-85b887cb9d-kjthl   1/1     Running   0          7m13s
pod/mondeploy-85b887cb9d-p6xbf   1/1     Running   0          6m59s
pod/mondeploy-85b887cb9d-zjmw5   1/1     Running   0          6m59s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   76m

NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/daemon   3         3         3       3            3           <none>          4s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mondeploy   4/4     4            4           9m35s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/mondeploy-746c94dc94   0         0         0       9m35s
replicaset.apps/mondeploy-85b887cb9d   4         4         4       7m13s
```

 You will notice that the `DaemonSets` runs on all the worker nodes without specifying the replicas
```
ansible@CTRL-01:~$ kubectl get pods -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
daemon-bz6n6                 1/1     Running   0          88s     172.16.89.197   wrk-01   <none>           <none>
daemon-llfdt                 1/1     Running   0          88s     172.16.19.67    wrk-02   <none>           <none>
daemon-r7z7z                 1/1     Running   0          88s     172.16.108.3    wrk-03   <none>           <none>
mondeploy-85b887cb9d-8z8n4   1/1     Running   0          8m37s   172.16.19.66    wrk-02   <none>           <none>
mondeploy-85b887cb9d-kjthl   1/1     Running   0          8m37s   172.16.108.2    wrk-03   <none>           <none>
mondeploy-85b887cb9d-p6xbf   1/1     Running   0          8m23s   172.16.89.195   wrk-01   <none>           <none>
mondeploy-85b887cb9d-zjmw5   1/1     Running   0          8m23s   172.16.89.196   wrk-01   <none>           <none>
```

## StatefulSets

Features that needed by stateful applications. Provide pod ordering and uniqueness. It also maintain sticky identifier for each Pods. You need StatefulSet to use in stateful application instead of deployments. Example of StatefulSet config can be found here: [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) Based on the config we noticed few settings:
1. `Kind` is `StatefulSet`
2. Headless service with `ClusterIP` set to none
3. Storage involve with `VolumeClaimTemplates` We can check the storage class using `kubectl get storageclass` If you dont have any storage configured, use this example and deploy your storage [StorageClass Example](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/storage/storageclass-low-latency.yaml)
We can deploy the stateful application by 1st checking the storage:
```
ansible@CTRL-01:~$ kubectl get storageclass
NAME          PROVISIONER                         RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
low-latency   csi-driver.example-vendor.example   Retain          WaitForFirstConsumer   true                   2m16s
```

Update the storage in the config and deploy:
```
ansible@CTRL-01:~$ grep storageClassName statefulset.yml
      storageClassName: "low-latency"
ansible@CTRL-01:~$ kubectl apply -f statefulset.yml
service/nginx created
statefulset.apps/web created
ansible@CTRL-01:~$ kubectl get all
NAME        READY   STATUS    RESTARTS   AGE
pod/web-0   0/1     Pending   0          3s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   110m
service/nginx        ClusterIP   None         <none>        80/TCP    3s

NAME                   READY   AGE
statefulset.apps/web   0/3     3s
```

## Running in Individual Pod
Has a lot of disadvantages when running individual pod, no redundancy and load-balacing and etc. Alway use `Deployments`, `DaemonSets` or `StatefulSets`

## Pod Initialization

Using `Init Containers` when you need to do preparation. Use `init containers` when there is a preliminary setup required. Example of `init container` that define Pod with 2 init containers [Init Container Example](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#init-containers-in-use) and edit as below:
```
ansible@CTRL-01:~$ cat myinit.yml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sleep', '20']
  - name: init-mydb
    image: busybox
    command: ['sleep', '20']
```

Apply the config, the main container wont run until the 2 init container ready:
```
ansible@CTRL-01:~$ kubectl get pods
NAME        READY   STATUS     RESTARTS   AGE
myapp-pod   0/1     Init:1/2   0          45s
web-0       0/1     Pending    0          11m
```

Confirmed that the main container runnning
```
ansible@CTRL-01:~$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
myapp-pod   1/1     Running   0          77s
web-0       0/1     Pending   0          11m
```

## Scalling Apps

`kubectl scale` command use to manually scale `Deployment`, `ReplicaSet` or `StatefulSet` Alternative we have `HorizontalPodAutoscaler`.
We scale down our previous deployment
```
ansible@CTRL-01:~$ kubectl scale deployment mondeploy --replicas=2
deployment.apps/mondeploy scaled
ansible@CTRL-01:~$ kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
mondeploy   2/2     2            2           67s
```

Set the deployment to use autoscaler with min and max replicas. There are may other options available use `kubectl autoscale -h | more` 
```
ansible@CTRL-01:~$ kubectl autoscale deployment mondeploy --min=5 --max=10
ansible@CTRL-01:~$ kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
mondeploy   2/2     2            2           2m51s
ansible@CTRL-01:~$ kubectl get hpa
NAME        REFERENCE              TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
mondeploy   Deployment/mondeploy   cpu: <unknown>/80%   5         10        5          41s
ansible@CTRL-01:~$ kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
mondeploy   5/5     5            5           3m46s
```
You can check the update config as below:
```
ansible@CTRL-01:~$ kubectl get deploy mondeploy -o yaml | grep replicas
  replicas: 5
  replicas: 5
```
