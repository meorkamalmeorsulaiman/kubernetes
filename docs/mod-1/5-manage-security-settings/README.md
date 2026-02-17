# Managing Security Settings

## Table of Contents

- [Security Context]()
- [Understanding User and Service Accounts]()
- [Understanding Role Based Access Control]()
- [API Access using default ServiceAccount]()


## API Access 

- Have to authenticate for API Access with certificate
- CA Infra built in with K8s
- Kubectl use certificate to access the API
- User need certificate to access API and it also has to bind with a role that managed by RBAC

## Security Context 

Defines privilege and access control settings for Pod or Container. More about security context settings on [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) To specify security settings for a Pod, you need to include the securityContext field in the Pod spec.Below is the example of security context:
```
cat <<EOF > secConPod.yaml
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
EOF
```

Then proceed to create the pod
```
kubectl apply -f secConPod.yaml
```

The the security context using below command:
```
kubectl exec -it security-context-demo -- ps
kubectl exec -it security-context-demo -- ls -l /data
kubectl exec -it security-context-demo -- id
```

## Understanding User and Service Accounts

K8s has 2 categories of users which is service accounts and normal users. The normal user is any user that present a valid certificate signed by the Cluster is considered authenticated and RBAC would determine whether the user is authorized to perform specific operation on a resource. On the other hand, the service accounts are users managed by the K8s API. It bound to a specific namespaces, and created automatically by the API server or manually through API calls. Serivce accounts are tied to a set of credentials stored as secrets, which are mounted into pods allowin in-cluster processes to talk to the K8s API. More on this in [K8s Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

## Understanding Role Based Access Control

Role is a collections of verbs (list, create and etc.) Role give access to resource(pods) Role binding connecting user and ServiceAccount (attaches to pod) with the Role. This create a RBAC environment. RBAC API declares 4 kinds of K8s object. They are Role, ClusterRole, RoleBinding and ClusterRoleBinding

## API Access using default ServiceAccount

Before starting let's test the API access from a Pod using below command it should failed - 403
```
kubectl run mypod --image=alpine -- sleep 3600
kubectl exec -it mypod -- apk add --update curl
kubectl exec -it mypod -- curl https://kubernetes/api/v1 --insecure
```

Now we set the service account token and use that as part of the http request. It should get some return
```
kubectl exec -it mypod -- sh
TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)
curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1 --insecure
```

Now try to use different api call and it should failed. This due to the default serviceAccount doesn't allow the get the pod details
```
curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/default/pods/ --insecure
```


## Setting up Role and RoleBinding

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

## Lab

Create a Role that allow for viewing Pods in default namespace and create a RoleBinding that allow authenticated user to use this role