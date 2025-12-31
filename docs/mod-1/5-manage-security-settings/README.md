# Managing Security Settings


## API Access 

- Have to authenticate for API Access with certificate
- CA Infra built in with K8s
- Kubectl use certificate to access the API
- User need certificate to access API and it also has to bind with a role that managed by RBAC

## Security Context 

Defines privilege and access control. Security context can be set at Pod or container level. Security context settings configured thru yml file. Example can be found in k8s documentations. Below is the example of security context:
```
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 2000
  volumes:
  - name: securevol
    emptyDir: {}
  containers:
  - name: sec-demo
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: securevol
      mountPath: /data/demo
    securityContext:
      allowPrivilegeEscalation: false
```

Apply the settings using `kubectl apply -f *.yml` and you can see the pod created
```
ansible@CTRL-01:~/cka$ kubectl apply -f security-context.yaml
pod/security-context-demo created
ansible@CTRL-01:~/cka$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
security-context-demo   1/1     Running   0          31s
```

Try to access the pod and check the processes. The properties shoul match the security context
```
ansible@CTRL-01:~/cka$ kubectl exec -it security-context-demo -- sh
~ $ ps
PID   USER     TIME  COMMAND
    1 1000      0:00 sleep 3600
    7 1000      0:00 sh
   13 1000      0:00 ps
~ $ cd /data/
/data $ ls -l
total 4
drwxrwsrwx    2 root     2000          4096 Dec 31 08:21 demo
/data $ id
uid=1000 gid=1000 groups=1000,2000
/data $ touch test.txt
touch: test.txt: Permission denied
/data $ exit
```

## Understanding User, ServiceAccount and API Access

Users doesn't created by k8s api for people to authenticate and authorize. They are obtained externally defined by:
- x.509 certificates
- OpenID-Based like AD

ServiceAccounts are used to authorized Pods to get access to specific API resources. Each Namespace has a ServiceAccount with the name `default`, this used by Pods to get minimal access to k8s resources. Additional ServiceAccount can be created to addtional access.

## Understanding Role Based Access Control

Role collections of verbs (list, create and etc.) Role give access to resource(pods) Role binding connecting user and ServiceAccount (attaches to pod) with the Role. This create a RBAC environment.

## Setting up RBAC for ServiceAccount

1st we need roles by using `kubectl create role [name]`
2st we need rolebinding using `kubectl create rolebinding [name]`
3rd we need ServiceAccount

We can try by first create a pod and check the ServiceAccount name as below:
```
ansible@CTRL-01:~/cka$ kubectl run mypod --image=alpine -- sleep 3600
pod/mypod created
ansible@CTRL-01:~/cka$ kubectl get pods mypod -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/containerID: ab32f45302e0896e0e96446032cd22acdb84c536811d5f9896049681ab26d63f
    cni.projectcalico.org/podIP: 172.16.19.65/32
    cni.projectcalico.org/podIPs: 172.16.19.65/32
  creationTimestamp: "2025-12-31T09:15:38Z"
  generation: 1
  labels:
    run: mypod
  name: mypod
  namespace: default
  resourceVersion: "30097"
  uid: cb4351a8-81b3-4b27-b27d-7626b3a48357
spec:
  containers:
  - args:
    - sleep
    - "3600"
    image: alpine
    imagePullPolicy: Always
    name: mypod
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-kcgpg
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: wrk-02
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-kcgpg
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2025-12-31T09:15:47Z"
    observedGeneration: 1
    status: "True"
    type: PodReadyToStartContainers
  - lastProbeTime: null
    lastTransitionTime: "2025-12-31T09:15:38Z"
    observedGeneration: 1
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2025-12-31T09:15:47Z"
    observedGeneration: 1
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2025-12-31T09:15:47Z"
    observedGeneration: 1
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2025-12-31T09:15:38Z"
    observedGeneration: 1
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: containerd://b03c48d44a70baefe04e859a77c63e2287d3350f3fb204ecff473f2d9e045e3d
    image: docker.io/library/alpine:latest
    imageID: docker.io/library/alpine@sha256:865b95f46d98cf867a156fe4a135ad3fe50d2056aa3f25ed31662dff6da4eb62
    lastState: {}
    name: mypod
    ready: true
    resources: {}
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2025-12-31T09:15:46Z"
    user:
      linux:
        gid: 0
        supplementalGroups:
        - 0
        - 1
        - 2
        - 3
        - 4
        - 6
        - 10
        - 11
        - 20
        - 26
        - 27
        uid: 0
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-kcgpg
      readOnly: true
      recursiveReadOnly: Disabled
  hostIP: 192.168.101.22
  hostIPs:
  - ip: 192.168.101.22
  observedGeneration: 1
  phase: Running
  podIP: 172.16.19.65
  podIPs:
  - ip: 172.16.19.65
  qosClass: BestEffort
  startTime: "2025-12-31T09:15:38Z"
```

Let check what the default ServiceAccount can do:
```
ansible@CTRL-01:~/cka$ kubectl get sa
NAME      SECRETS   AGE
default   0         3h53m
ansible@CTRL-01:~/cka$ kubectl exec -it mypod -- sh
/ # apk add --update curl
( 1/10) Installing brotli-libs (1.2.0-r0)
( 2/10) Installing c-ares (1.34.6-r0)
( 3/10) Installing libunistring (1.4.1-r0)
( 4/10) Installing libidn2 (2.3.8-r0)
( 5/10) Installing nghttp2-libs (1.68.0-r0)
( 6/10) Installing nghttp3 (1.13.1-r0)
( 7/10) Installing libpsl (0.21.5-r3)
( 8/10) Installing zstd-libs (1.5.7-r2)
( 9/10) Installing libcurl (8.17.0-r1)
(10/10) Installing curl (8.17.0-r1)
Executing busybox-1.37.0-r30.trigger
OK: 13.2 MiB in 26 packages
```

Then curl the k8s API
```
/ # curl https://kubernetes/api/v1 --insecure
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/api/v1\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
```

Assign a user to a variable that we can use to access the API
```
/ # TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)
/ # echo $TOKEN
eyJhbGciOiJSUzI1NiIsImtpZCI6IlgwYlBqb3ZPckhrWDdsWTY1TTJ1Q2ZIWUNrVXNaUmlIV2Nub1g1TktaN3cifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzk4NzA4NTM5LCJpYXQiOjE3NjcxNzI1MzksImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiMzc3NzEyZWUtNzA5MC00MzlmLTljYTktMjcwZTI2MjJiYzM5Iiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0Iiwibm9kZSI6eyJuYW1lIjoid3JrLTAyIiwidWlkIjoiMjk2MTQ3ZTItZjc5ZC00NzUxLWI2MzYtM2M2MDJkODRkNmIwIn0sInBvZCI6eyJuYW1lIjoibXlwb2QiLCJ1aWQiOiJjYjQzNTFhOC04MWIzLTRiMjctYjI3ZC03NjI2YjNhNDgzNTcifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRlZmF1bHQiLCJ1aWQiOiJlY2FiNzIwZi0xNDZlLTRlOTktYTdhYi1mYjEzYzRhN2M2ZWQifSwid2FybmFmdGVyIjoxNzY3MTc2MTQ2fSwibmJmIjoxNzY3MTcyNTM5LCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkZWZhdWx0In0.pdZbwWN9CsfMCmIn1H0Pm_lP-ntii_lq21aXGXtt2HwtFSfGUtMLJpW62kdeJ5dfsIzQ4t7WmPKvRvnMXoJ0aXFh7RGnaGpCPSdTCOndcBjgyz2hpCpNwGlB4dQgfpDxFKAnqn04cTKlJ-W_hhLQGsM-ZkVlzCE55fxLVLoawxbAwmL9HegKw0lBgCSMIQ7nvSWgMljNu1On7Ga5DlMesxarZJfot-19ybHcr4wG2ijI4I92kz4Aou_airyH2V56Pba-TX13iwKfXHUMmUV7Q8f3JQPsCsIZYg_IL2M8a7r4mEG24rTw0JQl5F3-ogAYRPB2F5rhQmB6tcYoM2snUw
```

Use that token to access the k8s API and should be able to get the API resource
```
/ # curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1 --insecure
<<Snippet>>
```

Access different API resource to see the namespaces, it failed with default ServiceAccount
```
/ # curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/default/pods/ --insecure
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:default:default\" cannot list resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}/ # exit
```

Now we can create new ServiceAccount and role. You can use `kubectl create role -h | less` to see the command options
```
ansible@CTRL-01:~/cka$ kubectl create sa mysa
serviceaccount/mysa created
ansible@CTRL-01:~/cka$ kubectl create role list-pods --resource=pods --verb=list
role.rbac.authorization.k8s.io/list-pods created
```

Now we create rolebinding and attach the created role and ServiceAccount
```
ansible@CTRL-01:~/cka$ kubectl create rolebinding list-pods --role=list-pods --serviceaccount=default:mysa
rolebinding.rbac.authorization.k8s.io/list-pods created
```

Now we create a pod config file as below with our new service account:
```
ansible@CTRL-01:~/cka$ cat mysapod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysapod
spec:
  serviceAccountName: mysa
  containers:
  - name: alpine
    image: alpine:3.9
    command:
    - "sleep"
    - "3600"
ansible@CTRL-01:~/cka$ kubectl apply -f mysapod.yaml
pod/mysapod created
```

Then try to use the pod to access the API resources with the 
```
ansible@CTRL-01:~/cka$ kubectl exec -it mysapod -- sh
/ # apk -- update
/ # apk add curl
/ # curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/ --insecure
<<snipped>>
```

Use differnet API source to get the list of pods:
```
/ # curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/default/pods --insecure
<<snippet>>
```