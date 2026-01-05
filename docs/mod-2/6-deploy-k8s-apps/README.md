# Deploying Kubernetes Application

### Topics
1. [Using Deployment](#URL)
2. [Running Agents with DeamonSets](#URL)
3. [Using StatefulSets](#URL)
4. [Individual Pos](#URL)
5. [Manage Pod initialization](#URL)
6. [Scaling application](#URL)
7. [Autoscale](#URL)
8. [Sidecar for logging](#URL)


## Using Deployments

Standard to run k8s deployment. Deployments responsible to start the pod. Deployments resources use replicaset to scale pod. Deployment offer rolling update feature. To deploy use `kubectl create deploy` 

## DeamonSets

A resource that starts application in each cluster node. It start necessary agents on all cluster nodes. We can use for user workloads. 
