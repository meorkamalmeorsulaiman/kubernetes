# Deploying Kubernetes Application

### Topics
1. [Using Deployment](#URL)
2. [Running Agents with DeamonSets](#URL)
3. [Using StatefulSets](#URL)
4. [Individual Pod](#URL)
5. [Manage Pod initialization](#URL)
6. [Scaling application](#URL)
7. [Autoscaler](#URL)
8. [Sidecar for logging](#URL)


## Deployments

Standard to run k8s deployment. Deployments responsible to start the pod. Deployments resources use replicaset to scale pod. Deployment offer rolling update feature. The command below create a deployment that
```
kubectl create deploy web-service --image=nginx --replicas=3
kubectl get all   
```

We can generate a manifest using command below
```
kubectl create deploy web-service --image=nginx --replicas=3 --dry-run=client -o yaml > web-service.yaml
cat web-service.yaml
kubectl apply -f web-service.yaml
kubectl get deploy web-service
```

You can scale the deployment as below:
```
kubectl scale deploy web-service --replicas=4
```

Or using the manifest
```
kubectl replace -f web-service.yaml
kubectl scale --replicas=2 -f web-service.yaml 
```

Update deployment image - rolling update deployment:
```
kubectl set image deploy web-service nginx=nginx:1.28
kubectl get pods
```

## DeamonSets

A resource that run application in each cluster node. A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. It start necessary agents on all cluster nodes. We can use for user workloads. DaemonSet has to be created from manifest. It content doesn't have much different that Deployment except kind should be DaemonSet and no replicas and strategy. Below is the sample from the same Deployment from previous section that has been converted into DaemonSet and once applied. The Pod will run on all the worker nodes without specifying the replicas
```
cat <<EOF > web-daemon.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: web-daemon
  name: web-daemon
spec:
  selector:
    matchLabels:
      app: web-daemon
  template:
    metadata:
      labels:
        app: web-daemon
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
EOF
kubectl apply -f web-daemon.yaml
kubectl get all
kubectl get daemonset web-daemon 
kubectl get pods -o wide
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

## Autoscaler - HorizontalPodAutoscaler (HPA)

It is an API resource that manages autocalling. It work based on usage statistics and rely from metrics-server. App will auto scale up or down if the threshold passed. Below configured the HPA:
```
ansible@CTRL-01:~$ kubectl top pods
NAME                         CPU(cores)   MEMORY(bytes)
mondeploy-d9d6c99f6-ldkkx    0m           4Mi
mondeploy-d9d6c99f6-m4sz5    0m           4Mi
mondeploy-d9d6c99f6-pwd7v    0m           4Mi
mondeploy-d9d6c99f6-qcpsc    0m           4Mi
mondeploy-d9d6c99f6-tjjt8    0m           4Mi
myapp-pod                    0m           0Mi
webstress-7d778f7544-d6pjj   0m           4Mi
webstress-7d778f7544-zhm4x   0m           4Mi
ansible@CTRL-01:~$ kubectl autoscale deployment webstress --min=2 --max=4 --cpu=80%
horizontalpodautoscaler.autoscaling/webstress autoscaled
ansible@CTRL-01:~$ kubectl get hpa
NAME        REFERENCE              TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
mondeploy   Deployment/mondeploy   cpu: <unknown>/80%   5         10        5          91m
webstress   Deployment/webstress   cpu: <unknown>/80%   2         4         0          5s
ansible@CTRL-01:~$ kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
mondeploy   5/5     5            5           95m
webstress   2/2     2            2           6m46s
```

It will autoscale after 5 mins by using k8s controller parameter or HPA setting `stabilizationWindowSeconds` using `kubectl edit hpa [deployment name]` as below
```
spec:
  behavior:
    scaleDown:
      policies:
      - periodSeconds: 15
        type: Percent
        value: 100
      selectPolicy: Max
      stabilizationWindowSeconds: 30
    scaleUp:
      policies:
      - periodSeconds: 15
        type: Pods
        value: 4
      - periodSeconds: 15
        type: Percent
        value: 100
      selectPolicy: Max
      stabilizationWindowSeconds: 0
```

If you want to set cluster wide, edit `/etc/kubernetes/manifests/kube-controller-manager.yaml` with `- --horizontal-pod-autoscaler-downscale-delay=30s` The staic pod will automatically updated.

## Sidecar or Multi-container

Multi-container pod setup that provide additional function to the pod, it is an additional container within the pod. Few categories:
1. Sidecar: provide additional functionalities
2. Ambassador: use a a proxy for external connection
3. Adapter: used to standadize or normalize main container output

K8s v1.29 and later, an init container with `restartPolicy` set to `Always` will consider as Sidecar container

### Setup a sidecar for logging

1. Create a pod that write a log
2. Create a shared volume with an emptyDir that mounted to the log location
3. Add Sidecar container that run nginx and mount the share volume on nginx html
4. Expose the service

### Getting the emptryDir shared volume config

Config can be found in [EmptryDir Example](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir-configuration-example) We need this as the base template for the volumeMount configs

### Getting the template for pod that write a log

Below is how we can generate and get the arguments to generate logs
```
ansible@CTRL-01:~$ kubectl run test --image=busybox --dry-run=client -o yaml -- sh "echo hello > /tmp/myfile"
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test
  name: test
spec:
  containers:
  - args:
    - sh
    - echo hello > /tmp/myfile
    image: busybox
    name: test
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

### Comple configs

The complete configs as below:
```
ansible@CTRL-01:~$ cat mysidecar.yml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar
spec:
  containers:
  - image: busybox
    name: logging
    volumeMounts:
    - mountPath: /messages
      name: cache-volume
    args:
    - sh
    - -c
    - echo hello > /messages/index.html
  - image: nginx
    name: sidecar
    volumeMounts:
    - mountPath: /usr/lib/nginx/html
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi
```

We noticed that the logging container mount it directory `/messages` to the shared volume named `cache-volume`. Then it will write into `/messages/index.html` as a log. The sidecar on the other hand mount it default http directory `/usr/lib/nginx/html` to the same shared volume with logging container. This will result in nginx displying the log content in http. Let's run and test container within the `sidecar`  pod.
```
ansible@CTRL-01:~$ kubectl apply -f mysidecar.yml
pod/sidecar created
ansible@CTRL-01:~$ kubectl exec -it sidecar -c exporter -- cat /usr/lib/nginx/html/index.html
hello
```

## Lab Practice

Tasks:
- Create a DaemonSet with name `labdaemon`
- Ensure the pod run on every worker in the cluster
