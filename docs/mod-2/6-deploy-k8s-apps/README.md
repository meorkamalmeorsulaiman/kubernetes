# Deploying Kubernetes Application

### Topics
1. [Using Deployment](#URL)
2. [Running Agents with DeamonSets](#URL)
3. [Using StatefulSets](#URL)
4. [Individual Pos](#URL)
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

A resource that starts application in each cluster node. It start necessary agents on all cluster nodes. We can use for user workloads. 
