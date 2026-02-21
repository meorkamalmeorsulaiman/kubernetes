# Deploying Kubernetes Application

### Topics
1. [Deployment](#URL)
2. [DeamonSet](#URL)
3. [StatefulSets](#URL)
4. [Individual Pod](#URL)
5. [Init Continers](#URL)
6. [Scalling application](#URL)
7. [Autoscaler](#URL)
8. [Sidecar for logging](#URL)


## Deployment

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

## DeamonSet

A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. It start necessary agents on all cluster nodes. We can use for user workloads. DaemonSet has to be created from manifest. It content doesn't have much different that Deployment except kind should be DaemonSet and no replicas and strategy. 

Below is the sample from the same Deployment from previous section that has been converted into DaemonSet and once applied. The Pod will run on all the worker nodes without specifying the replicas
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

Features that needed by stateful applications. Provide pod ordering and uniqueness. It also maintain sticky identifier for each Pods. You need StatefulSet to use in stateful application instead of Deployment. Example of StatefulSet config can be found here: [StatefulSet](https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/) Based on the config we noticed few settings:
1. `Kind` is `StatefulSet`
2. Headless service with `ClusterIP` set to none

Below is how to create a StatefulSet
```
cat <<EOF > web.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.21
        ports:
        - containerPort: 80
          name: web
EOF
kubectl apply -f web.yaml
kubectl get all
```

## Running in Individual Pod
Has a lot of disadvantages when running individual pod, no redundancy and load-balacing and etc. Alway use `Deployments`, `DaemonSets` or `StatefulSets`

## Init Containers

Init container used before the app containers are started. More details about init container [Init Container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#understanding-init-containers) Example below has 2 init containers what will first run and once up the actual container will run. Monitor the pod event and see the container started in sequence
```
cat <<EOF > initApp.yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-app
  labels:
    app.kubernetes.io/name: initApp
spec:
  containers:
  - name: app-container
    image: busybox
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-service
    image: busybox
    command: ['sleep', '20']
  - name: init-db
    image: busybox
    command: ['sleep', '20']
EOF
kubectl apply -f initApp.yaml
kubectl describe pod init-app
```

## Scalling Apps

`kubectl scale` command use to manually scale `Deployment`, `ReplicaSet` or `StatefulSet` Alternative we have `HorizontalPodAutoscaler`.
We scale down our previous deployment
```
kubectl scale deployment web-service --replicas=2
```

Set min and max replicas for a deployment
```
kubectl autoscale deployment web-service --min=5 --max=10
```

More about scaling option use command below
```
kubectl scale -h | less
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
