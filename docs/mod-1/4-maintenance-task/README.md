# Maintenance tasks

Steps for maintenance work within the cluster

## Metric servers for performance metrics

Not part of vanilla, have to install separately. Repo can be accessible from here: `https://github.com/kubernetes-sigs/metrics-server` Below is to install the metric server
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

We should see the new pod, but it's not ready
```
ansible@CTRL-01:~$ kubectl get pods -n kube-system | grep metrics
metrics-server-867d48dc9c-r5kn7            0/1     Running   0          14m
ansible@CTRL-01:~$ kubectl logs -n kube-system metrics-server-867d48dc9c-r5kn7 | tail -n 1
E1215 09:24:05.336642       1 scraper.go:149] "Failed to scrape node" err="Get \"https://192.168.101.21:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 192.168.101.21 because it doesn't contain any IP SANs" node="wrk-01"
```

We need to update the pod so that it allow insecure TLS using `kubectl edit -n kube-system deployments.apps metrics-server` by adding an argument
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

Wait until the new pod come up
```
ansible@CTRL-01:~$ kubectl get pods -n kube-system | grep metrics
metrics-server-6b77496796-lh5lp            1/1     Running   0          49s
```

Now we can use the metric server 
```
ansible@CTRL-01:~$ kubectl top node
NAME      CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)   
ctrl-01   203m         5%       758Mi           4%          
wrk-01    55m          1%       339Mi           2%
```

Other useful command `kubectl top pod -A`


##  Backing up the Etcd

