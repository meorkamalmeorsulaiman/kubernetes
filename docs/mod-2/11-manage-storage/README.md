# Managing Storage

## Understading K8s Storage Options

Pod Volumes aren't flexible and persistent volume claim (PVC) much better

## Accessing Storage with Pod Volumes

Pod Volumes are part of Pod spec and storage is hard coded in the Pod manifest. This options isn't flexible, Pod has other storage options. Also the ConfigMap can mounted as a Pod Volume. Let create a pod volume with 2 containers attached to the volume
```
apiVersion: v1
kind: Pod
metadata: 
  name: pod-vol
spec:
  containers:
  - name: centos1
    image: centos:7
    command:
      - sleep
      - "3600" 
    volumeMounts:
      - mountPath: /centos1
        name: test
  - name: centos2
    image: centos:7
    command:
      - sleep
      - "3600"
    volumeMounts:
      - mountPath: /centos2
        name: test
  volumes: 
    - name: test
      emptyDir: {}
```

We test to create in a one container and check the file in other container
```
ansible@CTRL01:~$ kubectl exec -it pod-vol -c centos1 -- touch /centos1/test-file.txt
ansible@CTRL01:~$ kubectl exec -it pod-vol -c centos2 -- ls /centos2/
test-file.txt
```

Above is the example of `emptyDir: {}` volume. All containers in the Pod can read and write the same files in the emptyDir volume, though that volume can be mounted at the same or different paths in each container. More about `emptyDir` in [EmptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)

## Configure Persistent Volume Storage

PersistentVolumes(PV) are API resources that represent specific storage. Pvs can created manually or automatically using `StorageClass` and storage provisioner. PV need a few config options:
- `storageClassName` - used as a selector label to allow PVCs to bind, or connect to s specific `StorageClass`
- `capacity` - the PV capacity in GiB
- `accessMode` - defines read/write or read-only access
- the specific type of storage

Pods don't connect to PVs directly but indirectly using PVC. One of the PV type is `local` where A `local` volume represents a mounted local storage device such as a disk, partition or directory. More about `local` PV [Local PV](https://kubernetes.io/docs/concepts/storage/volumes/#local) Below is the example:
```
ansible@CTRL01:~$ cat pv-local.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  hostPath:
    path: "/mydata"
```

Once applied, you should see the PV created. `hostPath` above state that the PV should attach to the local node `/mydata` PV by itself doesn't work. You need Persistent Volume Claim to use the PV.
```
ansible@CTRL01:~$ kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-local   1Gi        RWO            Retain           Available           local-storage   <unset>                          41s
```

## PersistentVolumeClaim

API resource that allow Pods to connect to any type of storage that is provided at a specific site. Below example of PVC manifest where `storageClassName` should match the PV that we want to attach
```
ansible@CTRL01:~$ cat pvc-claim.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
```

Validate as below
```
ansible@CTRL01:~$ kubectl get pv,pvc
NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pv-local   1Gi        RWO            Retain           Bound    default/example-pvc-claim   local-storage   <unset>                          2m31s

NAME                                      STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/example-pvc-claim   Bound    pv-local   1Gi        RWO            local-storage   <unset>                 82s
```

## Attaching Pod to a PVC

Now, we can create a Pod with a PVC attached to it. Below is the example
```
ansible@CTRL01:~$ cat pod-pvc.yaml 
kind: Pod
apiVersion: v1
metadata:
   name: pod-pvc
spec:
  volumes:
    - name: pvc-storage
      persistentVolumeClaim:
        claimName: example-pvc-claim
  containers:
    - name: container-pvc
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pvc-storage
```

Above tells us that, the container volume mounted to `pvc-storage`. The `pvc-storage` is refer to the spec volumes type above that is `persistentVolumeClaim` and the `claimName` refer to the name of a PVC that we created earlier. That is how it map back to the PVC from a Pod perspective. We can create a file inside the container
```
kubectl exec -it pod-pvc -- touch /usr/share/nginx/html/hellofile
```

Then we can traceback to the actualy PV by going to the actual worker node as below:
```
ansible@CTRL01:~$ kubectl get pod pod-pvc -o wide
NAME      READY   STATUS    RESTARTS   AGE     IP              NODE    NOMINATED NODE   READINESS GATES
pod-pvc   1/1     Running   0          8m35s   172.16.24.129   wrk01   <none>           <none>
ansible@CTRL01:~$ ssh wrk01
ansible@WRK01:~$ ls /mydata/
hellofile
```
>**Note:** to trace the directory for PV use `kubectl describe pv PV-NAME | grep Path`

## Volume Reclaim

The `persistentVolumeReclaimPolicy` is set on PV to set it behaviour once it no longer bound to PVC. `Retain` is the default value that it will be left in its current which is release, such that it can be manually reclaimed by an administrator. `Delete` means taht the PV will be deleted once it is released. `Recycle` means that the PV will be recycle back into the pool of unused PVs.


## ConfigMaps and Secret as Volumes

It is a volume that allow a container in a pod to mount a file to a volume. This is useful to mount a config file. 1st we create a the file 
```
echo hello world > index.html
```

Then create the configMap
```
kubectl create cm web-home-page --from-file=index.html
rm index.html
```

Validate the configMap content
```
kubectl describe cm web-home-page
```

Then create a webservice deployment
```
kubectl create deploy webservice --image=nginx
```

The add the configMap and volume mount in the pod container
```
kubectl edit deploy webservice

#Add below section under pod spec
    spec:
      volumes:
        - name: cmvol
          configMap:
            name: web-home-page
#Add section below under container spec
      containers:
      - image: nginx
        volumeMounts:
          - mountPath: /usr/share/nginx/html
            name: cmvol
```

We can check the file should be mounted to the `mountPath`
```
ansible@CTRL01:~$ kubectl exec -it webservice-78cf8c5995-cm7hd -- ls /usr/share/nginx/html
index.html
```

We can test the web home page by exposing the deployment
```
ansible@CTRL01:~$ kubectl expose deploy webservice --port=32080 --target-port=80
service/webservice exposed
ansible@CTRL01:~$ kubectl get pod
NAME                          READY   STATUS    RESTARTS      AGE
pod-vol                       2/2     Running   4 (10m ago)   130m
webservice-78cf8c5995-cm7hd   1/1     Running   0             117s
ansible@CTRL01:~$ kubectl get all
NAME                              READY   STATUS    RESTARTS      AGE
pod/pod-vol                       2/2     Running   4 (10m ago)   130m
pod/webservice-78cf8c5995-cm7hd   1/1     Running   0             2m2s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP     26h
service/webservice   ClusterIP   10.102.56.152   <none>        32080/TCP   10s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/webservice   1/1     1            1           7m40s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/webservice-78cf8c5995   1         1         1       2m2s
replicaset.apps/webservice-85655f6dfc   0         0         0       7m40s
ansible@CTRL01:~$ curl http://10.102.56.152:32080
hello world
```