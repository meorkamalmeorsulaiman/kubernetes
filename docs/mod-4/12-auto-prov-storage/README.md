# Auto-Provisioning Storage

## Storage Provisioners

External application for storage auto-provisioning.

## Example of NFS Storage Provisioners

What is needed?
1. Setup NFS storage - this has to be install on control and worker nodes
2. Use helm to install the NFS Storage Provisioner
3. Set the PVC that use NFS auto-provisioning
4. Deploy a pod that uses the PVC


### Setting up NFS

Install `nfs-server` on the control node or it can be a seperate server
```
sudo apt install nfs-server
```

Install `nfs-client` on the worker nodes
```
sudo apt install nfs-client
```

Setup the share directory on the server - on storage node
```
sudo mkdir /nfsexport
sudo sh -c  'echo "/nfsexport *(rw,no_root_squash)" > /etc/exports'
sudo systemctl  restart nfs-server
```

### Installing NFS Auto-Provisioner using Helm

For installation details, please see [NFS Subdirectory External Provisioner Helm Chart](https://artifacthub.io/packages/helm/nfs-subdir-external-provisioner/nfs-subdir-external-provisioner) Start with adding the helm repo:
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```

Proceed to install with supplied the NFS server address and storage location
```
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=ctrl01 \
    --set nfs.path=/nfsexport
```

Validate installation
```
kubectl get pods
kubectl get all
```
>**Note:** usually complex application will be installed in a seperate namespace

### Configuring PVC

Prior to create the PVC it must bind to a PV using a `storageClass`. In this case, while installing the auto-provisioner using helm. It will automatically create the `storageClass` as below:
```
ansible@CTRL01:~$ kubectl get storageclass
NAME         PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   5m17s
```

Now you can create a manifest for PVC that ties back to the `nfs-client` storageClass
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-auto-prov
spec:
  storageClassName: nfs-client # SAME NAME AS THE STORAGECLASS
  accessModes:
    - ReadWriteMany #  must be the same as PersistentVolume
  resources:
    requests:
      storage: 50Mi
```

Once applied, we should see the PV and PVC created
```
ansible@CTRL01:~$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-5653150f-1aa6-444c-9426-68e6a7b328b8   50Mi       RWX            Delete           Bound    default/nfs-pvc-auto-prov   nfs-client     <unset>                          21s
ansible@CTRL01:~$ kubectl get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
nfs-pvc-auto-prov   Bound    pvc-5653150f-1aa6-444c-9426-68e6a7b328b8   50Mi       RWX            nfs-client     <unset>                 24s
```

### Deploy a Pod using auto-provisioning PVC

Sample taken from previous lab. That mount container `/usr/share/nginx/html` to the shared pvc

```
kind: Pod
apiVersion: v1
metadata:
   name: pod-pvc
spec:
  volumes:
    - name: pvc-storage
      persistentVolumeClaim:
        claimName: nfs-pvc-auto-prov
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

Test the storage by creating a file
```
kubectl exec -it pod-pvc -- touch /usr/share/nginx/html/hellofile
```

Then validate that it should be created in the NFS mount
```
ansible@CTRL01:~$ ls /nfsexport/default-nfs-pvc-auto-prov-pvc-5653150f-1aa6-444c-9426-68e6a7b328b8/
hellofile
```

### Shared storage and replicas

1st we deploy a new pvc
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webservice-pvc-auto-prov
spec:
  storageClassName: nfs-client # SAME NAME AS THE STORAGECLASS
  accessModes:
    - ReadWriteMany #  must be the same as PersistentVolume
  resources:
    requests:
      storage: 50Mi
```

We can deploy multiple replicas
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webservice
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webservice
  template:
    metadata:
      labels:
        app: webservice
    spec:
      containers:
      - name: webservice-nginx
        image: nginx 
        ports:
        - containerPort: 80
        volumeMounts:
        - name: pvc-storage
          mountPath: "/usr/share/nginx/html" # Path inside the container where the volume will be mounted
      volumes:
      - name: pvc-storage # Name of the volume, matching the volumeMounts name
        persistentVolumeClaim:
          claimName: webservice-pvc-auto-prov # Name of the PVC created in the previous step
```

Expose the deployment
```
kubectl expose deploy webservice --type=NodePort --port=32080 --target-port=80
```

Now we can create a homepage from the nfs directory
```
echo "Homepage for nginx deployment stored in the nfs storage" > /nfsexport/default-webservice-pvc-auto-prov-pvc-[UUID]/index.html
```

We should test on all worker node - should have the same content
```
ansible@CTRL01:~$ curl wrk01:31969
Homepage for nginx deployment stored in the nfs storage
ansible@CTRL01:~$ curl wrk02:31969
Homepage for nginx deployment stored in the nfs storage
ansible@CTRL01:~$ curl wrk03:31969
Homepage for nginx deployment stored in the nfs storage
```

Let's make a change on the html, it should be reflected
```
echo "Update homepage from nfs storage" > /nfsexport/default-webservice-pvc-auto-prov-pvc-2fd2be1f-dcf3-453f-94f1-386f289b9cbb/index.html
```

Let's test
```
ansible@CTRL01:~$ curl wrk01:31969
Update homepage from nfs storage
ansible@CTRL01:~$ curl wrk02:31969
Update homepage from nfs storage
ansible@CTRL01:~$ curl wrk03:31969
Update homepage from nfs storage
```