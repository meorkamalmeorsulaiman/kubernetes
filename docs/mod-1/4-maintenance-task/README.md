# Maintenance tasks

Steps for maintenance work within the cluster. This topic divided into several sections:

#### Table of Contents

- [Metrics Server for Performance Metrics](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#metric-servers-for-performance-metrics)
- [Backing up Etcd](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#backing-up-the-etcd)
- [Restoring Etcd](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#backing-up-the-etcd)
- [Upgrade Control Node](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#backing-up-the-etcd)
- [Upgrade Worker Node](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#upgrade-worker-node)
- [K8s HA Cluster](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#k8s-ha-cluster)
- [K8s HA Cluster Configuration](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#k8s-ha-cluster-configuration)
- [Demo K8s HA Cluster Deployment](https://github.com/meorkamalmeorsulaiman/kubernetes/tree/main/docs/mod-1/4-maintenance-task#demo-k8s-ha-cluster-deployment)

## Metric Server for Performance Metrics

Not part of vanilla, have to install separately. metrics-server allow us to get k8s performance metrics. Repo can be accessible from here: [Metrics Server SiGS](https://github.com/kubernetes-sigs/metrics-server) Below is to install the metric server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Edit the deployment pod parameter so that it will ignore tls. Otherwise, it wont work
```
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --kubelet-insecure-tls
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        image: registry.k8s.io/metrics-server/metrics-server:v0.8.0

```

Once the pod come up. Then can use the metric server by checking the available commands:
```
kubectl top -h
kubectl top node
kubectl top pod
```

##  Backing up the Etcd

Below are the backup steps:
1. Ensure `etcdctl` and `etcdutl` binaries installed
2. Take a backup snapshot using `etcdctl`

To install the binaries, which can be download and extract from [etcd-github](https://github.com/etcd-io/etcd) Once download, extract the binary. The steps included in the download page. Below is the basic
```
wget https://github.com/etcd-io/etcd/releases/download/v3.6.7/etcd-v3.6.7-linux-amd64.tar.gz
mkdir /tmp/etcd
tar xzvf etcd-v3.6.7-linux-amd64.tar.gz -C /tmp/etcd --strip-components=1 --no-same-owner
sudo cp /tmp/etcd/etcdctl /usr/bin
sudo cp /tmp/etcd/etcdutl /usr/bin
etcdctl version
etcdutl version
```

- In older `etcdctl` use wrong API vesion by default, has to fix by using `sudo ETCDCTL_API=3 etcdctl ... snapshot save`
- To use `etcdctl` we need to spcify the etcd API endpoint including the cacert, cert, and key to be used
- The values can be found using `ps aux | grep etcd` or inside `/etc/kubernetes/pki/etcd/`
```
ansible@CTRL01:~$ sudo ps afx | grep etcd
   8364 pts/0    S+     0:00              \_ grep --color=auto etcd
   4145 ?        Ssl    0:08  \_ etcd --advertise-client-urls=https://192.168.101.11:2379 --cert-file=/etc/kubernetes/pki/etcd/server.crt --client-cert-auth=true --data-dir=/var/lib/etcd --feature-gates=InitialCorruptCheck=true --initial-advertise-peer-urls=https://192.168.101.11:2380 --initial-cluster=ctrl01=https://192.168.101.11:2380 --key-file=/etc/kubernetes/pki/etcd/server.key --listen-client-urls=https://127.0.0.1:2379,https://192.168.101.11:2379 --listen-metrics-urls=http://127.0.0.1:2381 --listen-peer-urls=https://192.168.101.11:2380 --name=ctrl01 --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt --peer-client-cert-auth=true --peer-key-file=/etc/kubernetes/pki/etcd/peer.key --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt --snapshot-count=10000 --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt --watch-progress-notify-interval=5s
   4153 ?        Ssl    0:20  \_ kube-apiserver --advertise-address=192.168.101.11 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

Before backing up, test the api endpoint using the following command:
```
sudo etcdctl --endpoints=[IP]:2379 --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key get / --prefix --keys-only
```

Once working, proceed to do the actual backup
```
sudo etcdctl --endpoints=localhost:2379 --cacert [Location] --cert [server cert] --key [server key] snapshot save /tmp/etcdbackup.db
```

Test the backup file
```
sudo etcdutl snapshot status /tmp/etcdbackup.db  --write-out=table
```

Validate that the pod and service running
```
ansible@CTRL01:~$ kubectl get all
NAME       READY   STATUS    RESTARTS   AGE
pod/vm01   1/1     Running   0          7m43s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP           10m
service/vm01         NodePort    10.101.39.23   <none>        32022:30381/TCP   7m23s
```

## Restoring Etcd backup

Procedures for restoring etcd:
1. Stop kubernestes service by moving the yaml configs somewhere else from `/etc/kubernetes/manifesets/*` 
2. Wait till the static pod gone, this can be validate using `sudo crictl ps`
3. After that, rename the etcd directory to something else `/var/lib/etcd`
4. Then proceed to restore using `sudo etcdctl snapshot restore /tmp/backup --data-dir /var/lib/etcd`
5. Proceed to move back the pod files into `/etc/kubernetes/manifests/*` and validate using `sudo crictl ps` to check the pod started

Start by delete the service to see the restoration different
```
kubectl delete svc vm01
kubectl get all
```

Let's move the pod files to other directory 
```
cd /etc/kubernetes/pki/etcd
sudo mv *.yaml ..
sudo mv /var/lib/etcd /var/lib/etcd-backup01
```

Validate that the pod for `etcd` and `kube-apiserver` gone
```
ansible@CTRL01:/etc/kubernetes/manifests$ sudo crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD                                       NAMESPACE
4bb7300420cdb       5e785d005ccc1       10 minutes ago      Running             calico-kube-controllers   0                   e2b7941cf7dbd       calico-kube-controllers-b45f49df6-w44k9   kube-system
ea82fb95017c4       52546a367cc9e       10 minutes ago      Running             coredns                   0                   dc70f459f8b46       coredns-66bc5c9577-wwlbq                  kube-system
7d85462772045       52546a367cc9e       10 minutes ago      Running             coredns                   0                   12669b963bb2d       coredns-66bc5c9577-vps6t                  kube-system
6b0972c8720d9       08616d26b8e74       10 minutes ago      Running             calico-node               0                   b7064915292b1       calico-node-vd287                         kube-system
2a4d9147e1d25       36eef8e07bdd6       11 minutes ago      Running             kube-proxy                0                   09e7792efcba1       kube-proxy-gvbfq                          kube-system
```

Once, proceed to restore the backup with
```
sudo etcdutl snapshot restore /tmp/etcdbackup.db --data-dir /var/lib/etcd
```

Check the library populated
```
ansible@CTRL01:/etc/kubernetes/manifests$ sudo ls /var/lib/etcd
member
```

Revert the static pod files
```
cd /etc/kubernetes/manifests/
sudo mv ../*.yaml .
ls
```

Then check the pods, etcd and kube-apiserver should be running and stable
```
ansible@CTRL01:/etc/kubernetes/manifests$ sudo crictl ps
CONTAINER           IMAGE               CREATED              STATE               NAME                      ATTEMPT             POD ID              POD                                       NAMESPACE
7010390af23b2       aec12dadf56dd       About a minute ago   Running             kube-scheduler            0                   6f0446a06f2b6       kube-scheduler-ctrl01                     kube-system
3c585838bf9f8       5826b25d990d7       About a minute ago   Running             kube-controller-manager   0                   7ae3a4a9adeb9       kube-controller-manager-ctrl01            kube-system
5c075f3ab5966       a3e246e9556e9       About a minute ago   Running             etcd                      0                   1fdfe72e3db50       etcd-ctrl01                               kube-system
c14bfc860b59d       aa27095f56193       About a minute ago   Running             kube-apiserver            0                   1035a1f7b5e35       kube-apiserver-ctrl01                     kube-system
94419f2b1ce67       5e785d005ccc1       About a minute ago   Running             calico-kube-controllers   2                   e2b7941cf7dbd       calico-kube-controllers-b45f49df6-w44k9   kube-system
ea82fb95017c4       52546a367cc9e       14 minutes ago       Running             coredns                   0                   dc70f459f8b46       coredns-66bc5c9577-wwlbq                  kube-system
7d85462772045       52546a367cc9e       14 minutes ago       Running             coredns                   0                   12669b963bb2d       coredns-66bc5c9577-vps6t                  kube-system
6b0972c8720d9       08616d26b8e74       14 minutes ago       Running             calico-node               0                   b7064915292b1       calico-node-vd287                         kube-system
2a4d9147e1d25       36eef8e07bdd6       14 minutes ago       Running             kube-proxy                0                   09e7792efcba1       kube-proxy-gvbfq                          kube-system
```

Check pods and svc which should be restored
```
ansible@CTRL01:/etc/kubernetes/manifests$ kubectl get all
NAME       READY   STATUS    RESTARTS   AGE
pod/vm01   1/1     Running   0          12m

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP           14m
service/vm01         NodePort    10.101.39.23   <none>        32022:30381/TCP   11m
```

## Upgrade control node

- Cluster can update to minor vesion only
- You need to update the K8s apt repo to the desire version
- 1st upgrade the `kubeadm` then upgrade the control node and finally the worker node

Control node upgrade steps:
1. Upgrade `kubeadm`
2. Check the available version using `kubeadm upgrade plan`
3. Run the upgrade using `kubeadm upgrade apply v[VERSION]`
4. Remove all current workloads from the node using `kubectl drain controlnode --ignore-daemonsets`
5. Upgrade and restart `kubelet` and `kubectl`
6. Bring back the control node using `kubectl uncordon [CONTROL NODE]`

Start by updating the repo as below (current is v1.33). Ref in https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/change-package-repository/
```
ansible@CTRL-01:~$ sudo cat /etc/apt/sources.list.d/kubernetes.list 
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
```

Run `sudo apt update` and check the available vesion
```
ansible@CTRL-01:~$ sudo apt-cache madison kubeadm
   kubeadm | 1.34.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
```

Proceed to upgrade using `sudo apt-mark unhold kubeadm && sudo apt-get update && sudo apt-get install -y kubeadm='1.34.1-*' && sudo apt-mark hold kubeadm`
Check `kubeadm` upgraded version - v1.34
```
ansible@CTRL-01:~$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"34", EmulationMajor:"", EmulationMinor:"", MinCompatibilityMajor:"", MinCompatibilityMinor:"", GitVersion:"v1.34.1", GitCommit:"93248f9ae092f571eb870b7664c534bfc7d00f03", GitTreeState:"clean", BuildDate:"2025-09-09T19:43:15Z", GoVersion:"go1.24.6", Compiler:"gc", Platform:"linux/amd64"}
```

Proceed to check cluster upgrade and available version using `sudo kubeadm upgrade plan`
```
ansible@CTRL-01:~$ sudo kubeadm upgrade plan
[preflight] Running pre-flight checks.
[upgrade/config] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade/config] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: 1.33.7
[upgrade/versions] kubeadm version: v1.34.1
[upgrade/versions] Target version: v1.34.3
[upgrade/versions] Latest version in the v1.33 series: v1.33.7

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE      CURRENT   TARGET
kubelet     ctrl-01   v1.33.7   v1.34.3
kubelet     wrk-01    v1.33.7   v1.34.3

Upgrade to the latest stable version:

COMPONENT                 NODE      CURRENT    TARGET
kube-apiserver            ctrl-01   v1.33.7    v1.34.3
kube-controller-manager   ctrl-01   v1.33.7    v1.34.3
kube-scheduler            ctrl-01   v1.33.7    v1.34.3
kube-proxy                          1.33.7     v1.34.3
CoreDNS                             v1.12.0    v1.12.1
etcd                      ctrl-01   3.5.24-0   3.6.4-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.34.3

Note: Before you can perform this upgrade, you have to update kubeadm to v1.34.3.

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```

Then apply the upgrade `sudo kubeadm upgrade apply v1.34.3` as provided with error where `kubeadm` version mismatched.
```
ansible@CTRL-01:~$ sudo kubeadm upgrade apply v1.34.3
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
[upgrade/preflight] Running preflight checks
[upgrade] Running cluster health checks
[upgrade/preflight] You have chosen to upgrade the cluster version to "v1.34.3"
[upgrade/versions] Cluster version: v1.33.7
[upgrade/versions] kubeadm version: v1.34.1
error: error execution phase preflight: the version argument is invalid due to these errors:

        - Specified version to upgrade to "v1.34.3" is higher than the kubeadm version "v1.34.1". Upgrade kubeadm first using the tool you used to install kubeadm

Can be bypassed if you pass the --force flag
To see the stack trace of this error execute with --v=5 or higher
```

Let's fix so that `kubeadm` runs on `v1.34.3`
```
ansible@CTRL-01:~$ sudo apt-mark unhold kubeadm && sudo apt-get update && sudo apt-get install -y kubeadm='1.34.3-*' && sudo apt-mark hold kubeadm
Canceled hold on kubeadm.
Hit:1 http://my.archive.ubuntu.com/ubuntu noble InRelease
Hit:2 http://my.archive.ubuntu.com/ubuntu noble-updates InRelease                                
Hit:3 http://my.archive.ubuntu.com/ubuntu noble-backports InRelease                              
Hit:4 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.34/deb  InRelease
Hit:5 http://security.ubuntu.com/ubuntu noble-security InRelease
Reading package lists... Done
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Selected version '1.34.3-1.1' (isv:kubernetes:core:stable:v1.34:pkgs.k8s.io [amd64]) for 'kubeadm'
The following packages will be upgraded:
  kubeadm
1 upgraded, 0 newly installed, 0 to remove and 9 not upgraded.
Need to get 12.5 MB of archives.
After this operation, 0 B of additional disk space will be used.
Get:1 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.34/deb  kubeadm 1.34.3-1.1 [12.5 MB]
Fetched 12.5 MB in 0s (25.3 MB/s)
debconf: delaying package configuration, since apt-utils is not installed
(Reading database ... 77216 files and directories currently installed.)
Preparing to unpack .../kubeadm_1.34.3-1.1_amd64.deb ...
Unpacking kubeadm (1.34.3-1.1) over (1.34.1-1.1) ...
Setting up kubeadm (1.34.3-1.1) ...
Scanning processes...                                                                                                                                     
Scanning candidates...                                                                                                                                    
Scanning linux images...                                                                                                                                  

Running kernel seems to be up-to-date.

Restarting services...

Service restarts being deferred:
 /etc/needrestart/restart.d/dbus.service
 systemctl restart systemd-logind.service
 systemctl restart unattended-upgrades.service

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
kubeadm set on hold.
ansible@CTRL-01:~$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"34", EmulationMajor:"", EmulationMinor:"", MinCompatibilityMajor:"", MinCompatibilityMinor:"", GitVersion:"v1.34.3", GitCommit:"df11db1c0f08fab3c0baee1e5ce6efbf816af7f1", GitTreeState:"clean", BuildDate:"2025-12-09T15:05:15Z", GoVersion:"go1.24.11", Compiler:"gc", Platform:"linux/amd64"}
```

Then rerun the `sudo kubeadm upgrade plan` with now `v1.34.3`
```
ansible@CTRL-01:~$ sudo kubeadm upgrade plan
[preflight] Running pre-flight checks.
[upgrade/config] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade/config] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: 1.33.7
[upgrade/versions] kubeadm version: v1.34.3
[upgrade/versions] Target version: v1.34.3
[upgrade/versions] Latest version in the v1.33 series: v1.33.7

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE      CURRENT   TARGET
kubelet     ctrl-01   v1.33.7   v1.34.3
kubelet     wrk-01    v1.33.7   v1.34.3

Upgrade to the latest stable version:

COMPONENT                 NODE      CURRENT    TARGET
kube-apiserver            ctrl-01   v1.33.7    v1.34.3
kube-controller-manager   ctrl-01   v1.33.7    v1.34.3
kube-scheduler            ctrl-01   v1.33.7    v1.34.3
kube-proxy                          1.33.7     v1.34.3
CoreDNS                             v1.12.0    v1.12.1
etcd                      ctrl-01   3.5.24-0   3.6.5-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.34.3

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```

Proceed to upgrade the control node `sudo kubeadm upgrade apply v1.34.3`
```
ansible@CTRL-01:~$ sudo  kubeadm upgrade apply v1.34.3
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
[upgrade/preflight] Running preflight checks
[upgrade] Running cluster health checks
[upgrade/preflight] You have chosen to upgrade the cluster version to "v1.34.3"
[upgrade/versions] Cluster version: v1.33.7
[upgrade/versions] kubeadm version: v1.34.3
[upgrade] Are you sure you want to proceed? [y/N]: y
[upgrade/preflight] Pulling images required for setting up a Kubernetes cluster
[upgrade/preflight] This might take a minute or two, depending on the speed of your internet connection
[upgrade/preflight] You can also perform this action beforehand using 'kubeadm config images pull'
W1215 13:44:25.474984  212953 checks.go:827] detected that the sandbox image "registry.k8s.io/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended to use "registry.k8s.io/pause:3.10.1" as the CRI sandbox image.
[upgrade/control-plane] Upgrading your static Pod-hosted control plane to version "v1.34.3" (timeout: 5m0s)...
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests1684343756"
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Renewing etcd-server certificate
[upgrade/staticpods] Renewing etcd-peer certificate
[upgrade/staticpods] Renewing etcd-healthcheck-client certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-12-15-13-44-58/etcd.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 1 Pods for label selector component=etcd
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Renewing apiserver certificate
[upgrade/staticpods] Renewing apiserver-kubelet-client certificate
[upgrade/staticpods] Renewing front-proxy-client certificate
[upgrade/staticpods] Renewing apiserver-etcd-client certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-12-15-13-44-58/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 1 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Renewing controller-manager.conf certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-12-15-13-44-58/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 1 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Renewing scheduler.conf certificate
[upgrade/staticpods] Moving new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backing up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2025-12-15-13-44-58/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This can take up to 5m0s
[apiclient] Found 1 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upgrade/control-plane] The control plane instance for this node was successfully upgraded!
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upgrade/kubeconfig] The kubeconfig files for this node were successfully upgraded!
W1215 13:48:58.188897  212953 postupgrade.go:116] Using temporary directory /etc/kubernetes/tmp/kubeadm-kubelet-config195283597 for kubelet config. To override it set the environment variable KUBEADM_UPGRADE_DRYRUN_DIR
[upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config195283597/config.yaml
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!
[upgrade/bootstrap-token] Configuring bootstrap token and cluster-info RBAC rules
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade] SUCCESS! A control plane node of your cluster was upgraded to "v1.34.3".

[upgrade] Now please proceed with upgrading the rest of the nodes by following the right order.
```

Proceed to upgrade the `kubelet` and `kubectl` by drain the node first using `kubectl drain control --ignore-daemonsets`
```
ansible@CTRL-01:~$ kubectl drain ctrl-01 --ignore-daemonsets
node/ctrl-01 cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/calico-node-tmnqj, kube-system/kube-proxy-k7jpn
evicting pod kube-system/calico-kube-controllers-7498b9bb4c-4zzpx
pod/calico-kube-controllers-7498b9bb4c-4zzpx evicted
node/ctrl-01 drained
```

Upgrade the `kubelet` and `kubectl` using 
```
sudo apt-mark unhold kubelet kubectl && sudo apt-get update && \
sudo apt-get install -y kubelet='1.34.3-*' kubectl='1.34.3-*' && \
sudo apt-mark hold kubelet kubectl
```

Then restart the service
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Next, uncorden the control node using `kubectl uncordon ctrl-01` and validate the version
```
ansible@CTRL-01:~$ kubectl uncordon ctrl-01
node/ctrl-01 uncordoned
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS   ROLES           AGE     VERSION
ctrl-01   Ready    control-plane   5h36m   v1.34.3
wrk-01    Ready    <none>          5h27m   v1.33.7
```

## Upgrade worker node

1. Updade apt source list
2. Upgrade the `kubeadm`
3. Proceed the upgrade `sudo kubeadm upgrade node`
```
ansible@WRK-01:~$ sudo kubeadm upgrade node
[upgrade] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
[upgrade/preflight] Running pre-flight checks
[upgrade/preflight] Skipping prepull. Not a control plane node.
[upgrade/control-plane] Skipping phase. Not a control plane node.
[upgrade/kubeconfig] Skipping phase. Not a control plane node.
W1215 14:02:27.311209  127163 postupgrade.go:116] Using temporary directory /etc/kubernetes/tmp/kubeadm-kubelet-config1295409584 for kubelet config. To override it set the environment variable KUBEADM_UPGRADE_DRYRUN_DIR
[upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config1295409584/config.yaml
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!
[upgrade/addon] Skipping the addon/coredns phase. Not a control plane node.
[upgrade/addon] Skipping the addon/kube-proxy phase. Not a control plane node.
ansible@WRK-01:~$ kubectl drain <node-to-drain^C--ignore-daemonsets
```
4. Then drain before upgrading `kubelet` and `kubectl` using `kubectl drain wrk-01 --ignore-daemonsets --delete-emptydir-data`
```
#On control node
kubectl drain wrk-01 --ignore-daemonsets
```

5. Proceed to upgrade `kubelet` and `kubectl` using
```
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.34.3-*' kubectl='1.34.3-*' && \
sudo apt-mark hold kubelet kubectl
```

6. Once finished, reload daemon and restart kubelet. 

7. Proceed to uncordon worker node and validate the patch work
```
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS   ROLES           AGE     VERSION
ctrl-01   Ready    control-plane   5h50m   v1.34.3
wrk-01    Ready    <none>          5h41m   v1.34.3
```

## K8s HA Cluster

### HA Options
HA Options:
1. Run the control plane and etcd on the same node - stacked control plane
2. Seperate the control plane and etcd on different nod - External etcd cluster

### HA Requirements
HA Requirments:
1. Load balancing with LB to distribute workload between the control nodes
2. LB feature can use external software
3. In the exam, LB setup isn't required

### K8s HA Cluster Configuration
Configuration:
1. Use HAProxy to load-balance port 6443 across the control nodes
2. Traffic forwarded to `kube-apiserver` port 6443
3. Use keepalived on all control node with a VIP
4. `kubectl` connect to the VIP instead of the `CTRL-01` IP address
5. Use provided script from the repo `setup-lb-ubuntu.sh` to setup the HA - run on one of the control node only

#### Keepalived and HAProxy Setup
Setting up the `keepalived` and `HAProxy` by running the script prior to initializing the cluster, update the VIP and the interface name accordingly. At the end you should be able to check the VIP
```
ansible@CTRL-01:~/cka$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether bc:24:11:fc:6f:e6 brd ff:ff:ff:ff:ff:ff
    altname enp0s18
    inet 192.168.101.11/24 brd 192.168.101.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 192.168.101.10/24 scope global secondary eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::be24:11ff:fefc:6fe6/64 scope link 
       valid_lft forever preferred_lft forever
```

#### Initializing K8s HA Cluster
Proceed to setup HA cluster with the following steps:
1. Run 
```
sudo kubeadm init --control-plane-endpoint "[VIP]:6443" --upload-certs
```
2. Configure the `kubectl` client
3. Setup CNI plugin with
```
kubectl apply -f  https://docs.projectcalico.org/manifests/calico.yaml
```
4. Copy and use the `kubeadm join` command, there will be 2 join commands - control and worker. Use option `--control-plane` for control node

Control node join command
```
  kubeadm join 192.168.101.10:6443 --token [TOKEN] --discovery-token-ca-cert-hash [CERT-HASH] --control-plane --certificate-key [CERT-KEY]
```

Worker node join command
```
kubeadm join [VIP]:6443 --token [TOKEN] --discovery-token-ca-cert-hash [CERT-HASH]
```

5. Verify the cluster nodes using `kubectl get nodes`


### Demo K8s HA Cluster Deployment

Setup the HA Cluster by starting on `CTRL-01` and should see the client setup and 2 join commands
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes running the following command on each as root:

  kubeadm join 192.168.101.10:6443 --token wgax9h.xsu2183yrdpon934 \
        --discovery-token-ca-cert-hash sha256:ce0d6e0ab6b99715b37054e0c67acc1767059bae9533ea8438168aa56a0be733 \
        --control-plane --certificate-key 77433bfa22546aad2bb784fbc201d3573a6726b375abf8f87ca37f1603d39745

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.101.10:6443 --token wgax9h.xsu2183yrdpon934 \
        --discovery-token-ca-cert-hash sha256:ce0d6e0ab6b99715b37054e0c67acc1767059bae9533ea8438168aa56a0be733 
```

Proceed to configure the `kubectl`
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

After that, proceed to install the CNI plugin
```
kubectl apply -f  https://docs.projectcalico.org/manifests/calico.yaml
```

Moving on, join the remaining control nodes into the cluster. Setup the kube client too. 
```
ansible@CTRL-02:~$ sudo kubeadm join 192.168.101.10:6443 --token wgax9h.xsu2183yrdpon934 \
        --discovery-token-ca-cert-hash sha256:ce0d6e0ab6b99715b37054e0c67acc1767059bae9533ea8438168aa56a0be733 \
        --control-plane --certificate-key 77433bfa22546aad2bb784fbc201d3573a6726b375abf8f87ca37f1603d39745
## Snippet ##
To start administering your cluster from this node, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

Validate the cluster nodes and check all controllers that
```
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS   ROLES           AGE     VERSION
ctrl-01   Ready    control-plane   7m18s   v1.34.3
ctrl-02   Ready    control-plane   3m4s    v1.34.3
ctrl-03   Ready    control-plane   26s     v1.34.3
```

Proceed to join the worker node in the cluster using the worker join command
```
ansible@WRK-01:~/cka$ sudo kubeadm join 192.168.101.10:6443 --token wgax9h.xsu2183yrdpon934 \
        --discovery-token-ca-cert-hash sha256:ce0d6e0ab6b99715b37054e0c67acc1767059bae9533ea8438168aa56a0be733 
[preflight] Running pre-flight checks
[preflight] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[preflight] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 1.001429479s
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

Validate the cluster nodes
```
ansible@CTRL-01:~$ kubectl get nodes
NAME      STATUS   ROLES           AGE     VERSION
ctrl-01   Ready    control-plane   11m     v1.34.3
ctrl-02   Ready    control-plane   6m50s   v1.34.3
ctrl-03   Ready    control-plane   4m12s   v1.34.3
wrk-01    Ready    <none>          74s     v1.34.3
```

Test by shutting down `ctrl-01` for testing and login to `ctrl-02`
```
ansible@ANS-CTRL:~/kubernetes$ ssh 192.168.101.10
ansible@CTRL-02:~$ kubectl get nodes
NAME      STATUS     ROLES           AGE     VERSION
ctrl-01   NotReady   control-plane   13m     v1.34.3
ctrl-02   Ready      control-plane   9m39s   v1.34.3
ctrl-03   Ready      control-plane   7m1s    v1.34.3
wrk-01    Ready      <none>          4m3s    v1.34.3
ansible@CTRL-02:~$ kubectl get pod -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-b45f49df6-sl42r   1/1     Running   0          10m
calico-node-4jkgm                         1/1     Running   0          6m6s
calico-node-8lxfh                         1/1     Running   0          10m
calico-node-gng5l                         1/1     Running   0          3m8s
calico-node-nrdw9                         1/1     Running   0          8m44s
coredns-66bc5c9577-hdp2c                  1/1     Running   0          12m
coredns-66bc5c9577-wjhnc                  1/1     Running   0          12m
etcd-ctrl-01                              1/1     Running   0          12m
etcd-ctrl-02                              1/1     Running   0          8m44s
etcd-ctrl-03                              1/1     Running   0          6m3s
kube-apiserver-ctrl-01                    1/1     Running   0          12m
kube-apiserver-ctrl-02                    1/1     Running   0          8m44s
kube-apiserver-ctrl-03                    1/1     Running   0          6m3s
kube-controller-manager-ctrl-01           1/1     Running   0          12m
kube-controller-manager-ctrl-02           1/1     Running   0          8m44s
kube-controller-manager-ctrl-03           1/1     Running   0          6m3s
kube-proxy-2nwmt                          1/1     Running   0          8m44s
kube-proxy-4rlwr                          1/1     Running   0          6m6s
kube-proxy-6kgj2                          1/1     Running   0          12m
kube-proxy-vpnkt                          1/1     Running   0          3m8s
kube-scheduler-ctrl-01                    1/1     Running   0          12m
kube-scheduler-ctrl-02                    1/1     Running   0          8m44s
kube-scheduler-ctrl-03                    1/1     Running   0          6m3s
ansible@CTRL-02:~$ 
```

## Lab Practice

Tasks:
- Backup Etcd and restore Restore Etcd

