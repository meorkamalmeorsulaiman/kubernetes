# Manage Scheduling


## Scheduling Process

This section explains the scheduling procees in K8s, it describes that process taken to a assigned Pods to a node.

### Understanding Scheduling

kube-scheduler will find a node to schedule new Pods. Node are selected according to requirement that may been set based on:
- Resource requirements
- AFfinity and anti-affinity
- Taints and tolerations and more

Scheduler first finds feasible nodes and score them, node that selected is the highest score. Once node was found, the scheduler notifies the API server in a process called binding.

### Scheduler to Kubelet

Once scheduler decided, kubelet will pick up and instruct CRI to pull the image of the container on specific node. In the end, container is created and started.
