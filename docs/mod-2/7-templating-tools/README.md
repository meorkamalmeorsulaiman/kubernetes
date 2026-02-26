# Using Templating Tools

## Table on Content

1. [Running Apps Using YAML Files]()
2. [Helm Package Manager]()
   - [Installing Helm]()
   - [Adding Bitnami Helm Chart Repository]()
   - [Finding Chart]()
   - [Install and Delete Chart]()
   - [Manage Installed Chart]()
3. [Generate a Manifest File from Helm Chart]()
   - [Generating the with custom value]()
4. [Kustomize]()
5. [Lab Practice]()

## Run Apps from YAML File

K8s apps often is a collection of resources. Using YAML file, consistency can be achieved. 

## Helm Package Manager

Helm is K8s package manager, use to streamline installing and managing K8s applications. Helm consist of `helm` tool that need to be installed and a chart. Helm chart is a package that contains:
- Description of a package
- One or more templates containing K8s manifest or resource file

### Installing Helm

Charts can be sotred locally or accessed remotely. Below is how to install helm binary by first download the binary. Prior to that confirm your architecture
```
arch
```

Binary can be obtained from helm github [Helm Releases](https://github.com/helm/helm/releases). Once downloaded, extract the binary files and move it:
```
wget https://get.helm.sh/helm-v4.1.1-linux-amd64.tar.gz
tar -zxf helm-v4.1.1-linux-amd64.tar.gz 
./linux-amd64/helm version
sudo mv linux-amd64/helm /usr/local/bin/
helm version
```

### Adding Bitnami Helm Chart Repository

Below is example to add Bitnami Helm Chart
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo list
```

### Finding Chart


Search exact chart within repo. This will display all the package available in any repo
```
helm search repo nginx 
```

Search exact chart with it available version
```
helm search repo nginx --versions
```

### Install and Delete Chart

Installing Chart mean we are installing the resources into the cluster. We start with updating the repo
```
helm repo update
```

Installing Grafana chart from Bitnami
```
helm install bitnami/grafana
helm list
kubectl get all
```

Delete chart
```
helm list
helm uninstall grafana-1772094697
```

### Manage Installed Chart

Get details about chart
```
helm show chart bitnami/grafana
```

Show all command related to the chart
```
helm show all bitnami/grafana
```

Get the chart details
```
helm get all grafana-1772094697
```

How currently installed applications
```
helm list
```

## Generate a Manifest File from Helm Chart

Using helm command to generate the template, you can then apply the template using `kubectl apply -f template.yaml` We try to generate a template for Argo CD First we add the Argo CD repo
```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm search repo argo/argo-cd
```

Now, let's generate the template. 
```
helm template my-argo-cd argo/argo-cd > argo-cd-template.yaml
cat argo-cd-template.yaml
```

Or for a specific version
```
helm template my-argo-cd argo/argo-cd --version 9.2.4 > argo-cd-template.yaml
cat argo-cd-template.yaml
```

Since this Chart required mandatory values to be set. Proceed to the next section for setting up the custom value

### Generating the with custom value

Generate actual value template for the Chart
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

Regenerate the manifest and deploy the chart
```
helm template my-argo-cd argo/argo-cd -f values.yaml > argo-cd-template.yaml
kubectl apply -f argo-cd-template.yaml 
kubectl get all
```

## Kustomize

kustomize use to apply changes to a set of resources. Filename should be `kustomization.yaml` and can be apply by using `kubectl apply -f kustomization.yaml` To use kustomize, we have to define the resource in the template as below:
```
resources:
  - deployment.yaml
  - service.yaml
namePrefix: test-
commonLabels:
  environment: testing
```

The `deployment.yaml` and `service.yaml` will define all other values. Now we apply the kustomize `-k` summarize and apply both deployment and service that stated in the kustomize file
```
kubectl apply -k .
```

We can see the pod started
```
NAME                                      READY   STATUS    RESTARTS   AGE
pod/test-nginx-friday20-dd867b57c-5cr24   1/1     Running   0          29s
pod/test-nginx-friday20-dd867b57c-m6wc2   1/1     Running   0          29s
pod/test-nginx-friday20-dd867b57c-xhcs6   1/1     Running   0          29s

NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/kubernetes            ClusterIP   10.96.0.1        <none>        443/TCP   11m
service/test-nginx-friday20   ClusterIP   10.107.237.240   <none>        80/TCP    29s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/test-nginx-friday20   3/3     3            3           29s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/test-nginx-friday20-dd867b57c   3         3         3       29s
```

We validate the label, should matched what stated in the kustomize
```
controlplane:~/cka/kustomize-demo$ kubectl get deploy --show-labels
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE     LABELS
test-nginx-friday20   3/3     3            3           3m11s   environment=testing,k8s-app=nginx-friday20
```

## Lab Practice

Use helm to install Vault Hashicorp application