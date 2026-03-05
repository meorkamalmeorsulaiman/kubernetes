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

Kustomize is a tool for customizing Kubernetes configurations. It has the following features to manage application configuration files:
- generating resources from other sources
- setting cross-cutting fields for resources
- composing and customizing collections of resources


Example below is a Based and Overlays concept. Where we have the based resource in a directory. Then we can have other version derive from that base rosource using kustomize. Let's start by creating the base resource that includes deployment and service
```
mkdir base
# Create a base/deployment.yaml
cat <<EOF > base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
EOF

# Create a base/service.yaml file
cat <<EOF > base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF
# Create a base/kustomization.yaml
cat <<EOF > base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

Once applied the deployment should be created
```
kubectl apply -k base/
kubectl get all
```

We should see the deployment and service created. Now, we create the overlay where we want to derive a new deployment called dev. An overlay directory only consist kustomize file that refers to other kustomization. In this case it should refer to the based kustomize. Let's create the overlay directory
```
mkdir dev
cat <<EOF > dev/kustomization.yaml
resources:
- ../base
namePrefix: dev-
EOF
```

Let's apply and see the deployement and service
```
kubectl apply -k dev/
kubectl get all
```

In the overlay kustomize we specify the prefix name. We can modify other values too. Refer to [Kustomize Feature List](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#kustomize-feature-list) Below example how to add label with adding the label to pod
```
kubectl delete -k dev/
cat <<EOF > dev/kustomization.yaml
resources:
- ../base
namePrefix: dev-
labels:
 - pairs:
    env: dev
   includeSelectors: false
EOF
kubectl get pod --show-labels
kubectl get deploy --show-labels
```

Set the selector to true will add label to the pod. This example shows that we can customize the overlay and set variety of variable without modifying the base resource.


## Lab Practice

Use helm to install Vault Hashicorp application