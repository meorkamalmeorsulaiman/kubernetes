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

## User, ServiceAccount and API Access



