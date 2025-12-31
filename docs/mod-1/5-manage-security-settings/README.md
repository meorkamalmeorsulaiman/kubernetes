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

Users doesn't created by k8s API for people to authenticate and authorize. They are obtained externally defined by:
- x.509 certificates
- OpenID-Based like AD

ServiceAccounts are used to authorized Pods to get access to specific API resources. Each Namespace has a ServiceAccount with the name `default`, this used by Pods to get minimal access to k8s resources. Additional ServiceAccount can be created to addtional access.

## Understanding Role Based Access Control

Role is a collections of verbs (list, create and etc.) Role give access to resource(pods) Role binding connecting user and ServiceAccount (attaches to pod) with the Role. This create a RBAC environment.

## Setting up RBAC for ServiceAccount

ServiceAccount is an identify in k8s that use to authenticate to API server or implementing security policies.

We can try by first create a pod and check the ServiceAccount name as below:
```
ansible@CTRL-01:~/cka$ kubectl run mypod --image=alpine -- sleep 3600
pod/mypod created
ansible@CTRL-01:~/cka$ kubectl get pods mypod -o yaml
<<snippet>>
  serviceAccount: default
  serviceAccountName: default
<<snippet>>
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

## ClusterRoles and ClusterRoleBindings

Role have a namespace scope while ClusterRole apply to the entire cluster. It work similar to what Roles do. To provide access to ClusterRoles, use user or SerivceAccount and attach it with ClusterRoleBinding. By default ClusterRole and ClusterRoleBinding exist and can find it using `kubectl get clusterrole` or `kubectl get clusterrolebinding`

## Setting up RBAC for Users

User account consist of authorized certificate, to create a user account:
- Create certificate key pair
- Create CSR
- Sign the certificate
- Create a configuration file that uses these keys to access the k8s cluster
- Create an RBAC Role
- Create an RBAC RoleBinding

This section not covered in CKA

Below steps to create user account:
1 - Create namespace `kubectl create ns NAME`
```
ansible@CTRL-01:~$ kubectl create ns students
namespace/students created
ansible@CTRL-01:~$ kubectl create ns staff
namespace/staff created
ansible@CTRL-01:~$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```

2 - Create user account and make sure it setup with key material using linux
```
ansible@CTRL-01:~$ sudo useradd -m -G sudo -s /bin/bash anna
ansible@CTRL-01:~$ sudo passwd anna
New password:
Retype new password:
passwd: password updated successfully
```

3 - Create private key
```
anna@CTRL-01:~$ openssl genrsa -out anna.key 2048
anna@CTRL-01:~$ ls -l
total 4
-rw------- 1 anna anna 1704 Dec 31 10:05 anna.key
```

4 - Create CSR
```
anna@CTRL-01:~$ openssl req -new -key anna.key -out anna.csr -subj "/CN=anna/O=k8s"
anna@CTRL-01:~$ ls -l
total 8
-rw-rw-r-- 1 anna anna  903 Dec 31 10:06 anna.csr
-rw------- 1 anna anna 1704 Dec 31 10:05 anna.key
```

5 - Sign the certificate
```
anna@CTRL-01:~$ sudo openssl x509 -req -in anna.csr -CA /etc/kubernetes/pki/ca.crt
Certificate request self-signature ok
subject=CN = anna, O = k8s
Could not read CA private key from /etc/kubernetes/pki/ca.crt
```

6 - Use the Signed CSR to create the PKI certificate
```
anna@CTRL-01:~$ sudo openssl x509 -req -in anna.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out anna.crt -days 1800
Certificate request self-signature ok
subject=CN = anna, O = k8s
anna@CTRL-01:~$ ls -l
total 12
-rw-r--r-- 1 root root 1005 Dec 31 10:10 anna.crt
-rw-rw-r-- 1 anna anna  903 Dec 31 10:06 anna.csr
-rw------- 1 anna anna 1704 Dec 31 10:05 anna.key
```

7 - Create config directory for the new user 
```
anna@CTRL-01:~$ mkdir /home/anna/.kube
anna@CTRL-01:~$ sudo cp -i /etc/kubernetes/admin.conf .kube/config
anna@CTRL-01:~$ sudo chown -R anna:anna .kube
```

8 - Update the k8s credentials file for the new user
```
anna@CTRL-01:~$ kubectl config set-credentials anna --client-certificate=anna.crt --client-key=anna.key
User "anna" set.
```

Set context
```
anna@CTRL-01:~$ kubectl config set-context ann-context --cluster=kubernetes --namespace=staff --user=anna
Context "ann-context" created.
anna@CTRL-01:~$ kubectl config use-context anna-context
Switched to context "anna-context".
```

Test the access
```
anna@CTRL-01:~$ kubectl get pods
Error from server (Forbidden): pods is forbidden: User "anna" cannot list resource "pods" in API group "" in the namespace "staff"
anna@CTRL-01:~$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         anna-context                  kubernetes   anna               staff
          kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```

9 - Setup RBAC
```
ansible@CTRL-01:~$ kubectl create role staff -n staff --verb=get,list,watch,create,update,patch,delete --resource=deployment,replicasets,pods
role.rbac.authorization.k8s.io/staff created
```

10 - Bind the role
```
ansible@CTRL-01:~$ kubectl create rolebinding -n staff staff-role-binding --user=anna --role=staff
rolebinding.rbac.authorization.k8s.io/staff-role-binding created
ansible@CTRL-01:~$ su - anna
```

Test
```
anna@CTRL-01:~$ kubectl create deploy nginx --image=nginx
deployment.apps/nginx created
anna@CTRL-01:~$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-66686b6766-6tdzc   1/1     Running   0          14s
```

Options, assigned anna to role in default namespace with view-only permission
```
ansible@CTRL-01:~$ kubectl create role viewers -n default --verb=list,get,watch --resource=deployments,replicasets,pods
role.rbac.authorization.k8s.io/viewers created
ansible@CTRL-01:~$ kubectl create rolebinding viewers --user=anna --role=viewers
rolebinding.rbac.authorization.k8s.io/viewers created
ansible@CTRL-01:~$ su - anna
Password:
anna@CTRL-01:~$ kubectl get all -n default
NAME                        READY   STATUS    RESTARTS      AGE
pod/mypod                   1/1     Running   2 (30m ago)   150m
pod/mysapod                 1/1     Running   2 (13m ago)   134m
pod/security-context-demo   1/1     Running   3 (24m ago)   3h25m
Error from server (Forbidden): replicationcontrollers is forbidden: User "anna" cannot list resource "replicationcontrollers" in API group "" in the namespace "default"
Error from server (Forbidden): services is forbidden: User "anna" cannot list resource "services" in API group "" in the namespace "default"
Error from server (Forbidden): daemonsets.apps is forbidden: User "anna" cannot list resource "daemonsets" in API group "apps" in the namespace "default"
Error from server (Forbidden): statefulsets.apps is forbidden: User "anna" cannot list resource "statefulsets" in API group "apps" in the namespace "default"
Error from server (Forbidden): horizontalpodautoscalers.autoscaling is forbidden: User "anna" cannot list resource "horizontalpodautoscalers" in API group "autoscaling" in the namespace "default"
Error from server (Forbidden): cronjobs.batch is forbidden: User "anna" cannot list resource "cronjobs" in API group "batch" in the namespace "default"
Error from server (Forbidden): jobs.batch is forbidden: User "anna" cannot list resource "jobs" in API group "batch" in the namespace "default"
```