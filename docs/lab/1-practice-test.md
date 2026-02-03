# Practice Lab - Exam 1

## Create Cluster

- Deploy a cluster with old version with one control node.
- Join a worker node.

## Static Pod

- Create one static pod using nginx image

## Metrics Server

- Install metrics server 

## Backup and Restore etcd

- Deploy a virutal machine `vm01` with ubuntu image and ensure ssh install
- Expose ssh service for `vm01` with port 32022
- Backup etcd by creating a snapshot
- Delete the ssh service for `vm01`
- Restore etcd, ensure that the previous state restored

## Upgrades control and worker node

- Update control and worker to latest v1.34

