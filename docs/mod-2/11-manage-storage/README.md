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