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
