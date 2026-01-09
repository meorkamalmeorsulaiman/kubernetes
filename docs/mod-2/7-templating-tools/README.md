# Using Templating Tools

1. [Running Apps Using YAML Files]()
2.  [Helm Package Manager]()
3.  [Create Template from Helm Chart]()
4.  [Manage Apps with Helm]()
5.  [Kustomize]()
6.  [Lab Practice]()

## Run Apps from YAML File

K8s apps often is a collection of resources. Using YAML file, consistency can be achieved. 

## Helm Package Manager

Helm is K8s package manager, use to streamline installing and managing K8s applications. Helm consist of `helm` tool that need to be installed and a chart. Helm chart is a package that contains:
- Description of a package
- One or more templates containing K8s manifest file

### Installing Helm

Charts can be sotred locally or accessed remotely. Below is how to install helm binary by first download the binary. Prior to that confirm your architechture
```
controlplane:~$ arch
x86_64
```

Binary can be obtained from helm github `https://github.com/helm/helm/releases`. Once downloaded, extract the binary files and move it:
```
controlplane:~/linux-amd64$ ./helm version
version.BuildInfo{Version:"v4.0.4", GitCommit:"8650e1dad9e6ae38b41f60b712af9218a0d8cc11", GitTreeState:"clean", GoVersion:"go1.25.5", KubeClientVersion:"v1.34"}
controlplane:~/linux-amd64$ sudo mv helm  /usr/local/bin/
controlplane:~/linux-amd64$ cd
controlplane:~$ helm version
version.BuildInfo{Version:"v4.0.4", GitCommit:"8650e1dad9e6ae38b41f60b712af9218a0d8cc11", GitTreeState:"clean", GoVersion:"go1.25.5", KubeClientVersion:"v1.34"}
```

### Using Helm

Below are the steps for using Helm

#### Adding repository

```
controlplane:~$ helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
controlplane:~$ helm repo list
NAME                    URL                                                
kubernetes-dashboard    https://kubernetes.github.io/dashboard/            
metrics-server          https://kubernetes-sigs.github.io/metrics-server/  
kubelet-csr-approver    https://postfinance.github.io/kubelet-csr-approver/
rimusz                  https://charts.rimusz.net                          
bitnami                 https://charts.bitnami.com/bitnami 
```

#### Finding Chart

Use below command to search thru the repo
```
helm search repo bitnami
```

Search exact package
```
helm search repo nginx 
```

Search exact package with it available version
```
helm search repo nginx --versions
```

#### Installing Chart

Update the repo
```
helm repo update
```

Installing chart
```
helm install [name]/[chart]
```

Installing bitname mysql
```
helm install bitnami/mysql --generate-name
```

#### Manage the installed app

```
kubectl get all
```

Get details about chart
```
helm show chart bitnami/mysql
```

Show all command related to the chart
```
helm show all bitnami/mysql
```

Get the chart details
```
helm get all mysql-1767946255
```

How currently installed applications
```
helm list
```

Get the status using `kubectl`
```
kubectl describe pods mysql-1767946255-0
```

## Generate a Template from Helm Chart

Using helm command to generate the template, you can then apply the template using `kubectl apply -f template.yaml` 

### Installing argo-cd using helm template

#### Setting up helm for argo-cd

Adding repo
```
helm repo add argo https://argoproj.github.io/argo-helm
```

Repo update
```
helm repo update
```

Search chart
```
helm repo argo/argo-cd
```

#### Generate the template

Command format
```
helm template [application name] [repo name]/[chart name] --version [version] > template.yaml
```

Generate actual template
```
helm template my-argo-cd argo/argo-cd --version 9.2.4 > argo-cd-template.yaml
```

#### Generating the template with custom value

Command format for getting the chart values
```
helm show values [repo name]/[chart name] > template-value.yaml
```

Generate actual value template
```
helm show values  argo/argo-cd > values.yaml
```

Then edit the required mandatory value - service config, clusterIP
```
  ## Server service configuration
  service:
    # -- Server service annotations
    annotations: {}
    # -- Server service labels
    labels: {}
    # -- Server service type
    type: NodePort
    # -- Server service http port for NodePort service type (only if `server.service.type` is set to "NodePort")
    nodePortHttp: 30080
    # -- Server service https port for NodePort service type (only if `server.service.type` is set to "NodePort")
    nodePortHttps: 30443
    # -- Server service http port
    servicePortHttp: 80
```

Generate the template with custom value
```
helm template my-argocd argo/argo-cd -f values.yaml > argo-cd-template.yaml
```

#### Deploy the app using kubectl

```
kubectl apply -f argo-cd-template.yaml 
```

#### Remove app

```
kubectl delete -f argo-cd-template.yaml 
```


