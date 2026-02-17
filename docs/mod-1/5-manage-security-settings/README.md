# Managing Security Settings

## Table of Contents

- [Security Context]()
- [Understanding User and Service Accounts]()
- [Understanding Role Based Access Control]()
- [API Access using default ServiceAccount]()
- [Setting up Role and RoleBinding]()
- [Setting up Role and RoleBinding]()
- [Setting up RBAC for Users]()

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

We start by creating Service Account
```
kubectl create sa list-pods
```

Then we define the RBAC API Object - role and roleBinding. The roleBinding binds the role and the service account
```
kubectl create role list-pods --resource=pods --verb=list
kubectl create rolebinding list-pods --role=list-pods --serviceaccount=default:list-pods
```

Now we create a pod config file as below with our new service account:
```
cat <<EOF > pod-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-sa
spec:
  serviceAccountName: list-pods
  containers:
  - name: alpine
    image: alpine:3.9
    command:
    - "sleep"
    - "3600"
EOF
kubectl apply -f pod-sa.yaml
```

We test the API access of listing the pods and should be able to do so
```
kubectl exec -it pod-sa -- sh
apk add --update curl
TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)
curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/ --insecure
curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/namespaces/default/pods --insecure
```

## ClusterRoles and ClusterRoleBindings

Role have a namespace scope while ClusterRole apply to the entire cluster. It work similar to what Roles do. To provide access to ClusterRoles, use user or SerivceAccount and attach it with ClusterRoleBinding. By default ClusterRole and ClusterRoleBinding exist and can find it using `kubectl get clusterrole` or `kubectl get clusterrolebinding`

## Setting up RBAC for Users

This section is an optional that provide a step to create user account to authenticate and authorized to the cluster. Several steps define as below:
1. Create certificate key pair
2. Create CSR
3. Sign the certificate
4. Setup kubectl and add new pki into the cluster
5. Create an RBAC Role RBAC RoleBinding

> **Notes:** This section not covered in CKA

Create a namespace and add new user in linux environment
```
kubectl create ns staff
sudo useradd -m -G sudo -s /bin/bash it-administrator
sudo passwd it-administrator
```

Create Certificate Key Pair and Sign Certificate
```
su it-administrator
cd
openssl genrsa -out it-administrator.key 2048
openssl req -new -key it-administrator.key -out it-administrator.csr -subj "/CN=it-administrator/O=k8s"
sudo openssl x509 -req -in it-administrator.csr -CA /etc/kubernetes/pki/ca.crt 
sudo openssl x509 -req -in it-administrator.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out it-administrator.crt -days 1800
```

Setup kubectl and add new pki into the cluster using new user account
```
mkdir ~/.kube
sudo cp -i /etc/kubernetes/admin.conf .kube/config
sudo chown -R it-administrator:it-administrator .kube
kubectl config set-credentials it-administrator --client-certificate=it-administrator.crt --client-key=it-administrator.key 
kubectl config set-context staff-context --cluster=kubernetes --namespace=staff --user=it-administrator
kubectl config get-contexts
```

Create Role and RoleBinding using default control node account
```
su ansible
kubectl create role -n staff staff-role --verb=get,list,watch,create,update,patch,delete --resource=deployment,replicasets,pods
kubectl create rolebinding -n staff staff-binding --user=it-administrator --role=staff-role
```

Proceed to test 
```
su - it-administrator
kubectl get all
```

In case of assigning a view-only role, below is the verb settings and bind again with the user
```
kubectl create role viewers -n default --verb=list,get,watch --resource=deployments,replicasets,pods
```

## Lab

Create a Role that allow for viewing Pods in default namespace and create a RoleBinding that allow authenticated user to use this role