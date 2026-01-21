# Manage Application Access

## Kubernetes Networking

### Basic Networking Components

Network communications:
1. Between containers implemented as IPC - inter process communication. Not real networking
2. Between Pods implemented by network plugins
3. Between Pods and Services implemented by Service resources
4. Between external users and Services implemented by Services with ingress or gateway API

Incoming traffic manage by ingress but now replaced by gateway API. Both cant run on the same machine.
